# 01 — 数据库迁移 + 声纹库表

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

为 FunASR + 声纹识别 + 摘要服务创建所需的数据库表，作为后续所有 issue 的基础设施。

**关键架构决策 (grill-with-docs 确认)**: 

- FunASR (:8178) 独立 SQLite (声纹库)
- FastAPI (:5167) 独立 SQLite (meetings + transcripts + summaries)，**是 meetings 数据的唯一归属**
- Rust 不做自己的 SQLite。崩溃恢复状态用 JSON sidecar 文件 (`~/.meetily/recovery/{meeting_id}.json`)

### FunASR 侧 SQLite（`services/funasr/data.db`）

**新表 `speaker_embeddings`**：

| 列 | 类型 | 说明 |
|---|---|---|
| `id` | INTEGER PRIMARY KEY | 自增 ID |
| `name` | TEXT NOT NULL UNIQUE | 说话人姓名 |
| `embedding` | BLOB NOT NULL | CAM++ 256 维 float32 向量（二进制存储） |
| `created_at` | TEXT NOT NULL | ISO 8601 时间戳 |
| `updated_at` | TEXT NOT NULL | ISO 8601 时间戳 |

启动时幂等创建 (`CREATE TABLE IF NOT EXISTS`)。仅 Python `services/funasr/` 管理，Rust 不直连此文件。

**新表 `config`** (K-V 配置)：

| 列 | 类型 | 说明 |
|---|---|---|
| `key` | TEXT PRIMARY KEY | 配置项名 |
| `value` | TEXT NOT NULL | 配置值 |

初始数据：`INSERT OR IGNORE INTO config (key, value) VALUES ('similarity_threshold', '0.7')`。
启动时幂等创建。FunASR 启动时读取阈值，提供 `GET /config` + `POST /config` 端点。

### FastAPI 侧 SQLite（`meeting_minutes.db`）

**`meetings` 表新增字段** (migration, 幂等: `ALTER TABLE ... ADD COLUMN` 捕获 `OperationalError`)：

| 列 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `transcription_status` | TEXT | `'recording'` | `recording` / `complete` / `partial` |
| `audio_file_path` | TEXT | NULL | 完整原始录音路径（容灾用） |
| `expected_speakers` | INTEGER | 3 | 会议前用户输入的预期说话人数 k（离线重处理时读取） |
| `transcript_json` | TEXT | NULL | segments JSON 数组，带 speaker 字段。会后命名完成后写入 |
| `summary_json` | TEXT | NULL | Ollama 摘要输出 (Markdown)，用户手动触发生成后写入 |

> **注意**: 这些 migration 加在 FastAPI 侧 (`backend/app/db.py`)，而非 Rust 侧。Rust 不持有自己的 meeting 数据库，通过 FastAPI API 读写 meeting 数据。

### Rust 侧崩溃恢复 sidecar（JSON 文件）

录音开始时创建 `~/.meetily/recovery/{meeting_id}.json`：

```json
{
  "meeting_id": "abc123",
  "audio_file_path": "/recordings/abc123.wav",
  "expected_speakers": 3,
  "status": "recording",
  "raw_segments": null
}
```

字段说明：
- `raw_segments`: `/session/end` 返回后立即写入 (含临时聚类标签的 segments 数组)；命名完成 + save-transcript 成功后置 null 或删文件
- `named_segments`: `save-transcript` 失败时写入 (含最终姓名的 segments 数组)

**状态转换**：
- 录音开始 → `status: "recording"`, `raw_segments: null`
- `/session/end` 返回 → 写入 `raw_segments: [...]`
- 录音中每 5 分钟 → 增量追加 `raw_segments` (已有 segments 追加写入，防 Tauri 崩溃)
- `/session/end` 返回后 → 最终覆盖 `raw_segments: [...]` (全量)
- 断路器打开 → `status: "partial"`
- 命名完成 + save-transcript 成功 → 保留文件 (录音永不删)，清空 raw_segments/named_segments
- save-transcript 网络失败 → 写入 `named_segments: [...]`，启动时扫描重试
- Rust 启动时扫描 `raw_segments` 非 null → 重新 emit session-ended 弹命名 UI
- Rust 启动时扫描 `named_segments` 非 null → 自动重试 save-transcript
- Rust 启动时扫描 `status: "partial"` → 推送恢复横幅 (离线重处理)

### 涉及的代码区域

- **Python** `services/funasr/db.py` — 新建，管理 `speaker_embeddings` 表的 CRUD
- **Python** `backend/app/db.py` — migration：添加 `transcription_status`、`audio_file_path`、`expected_speakers`、`transcript_json`、`summary_json` 列到 meetings 表
- **Rust** — JSON sidecar 读写逻辑 (`recovery.rs`)，在 Issue B 阶段 2 引入

## Acceptance criteria

- [ ] FunASR 启动时自动创建 `speaker_embeddings` 表（幂等：`IF NOT EXISTS`）
- [ ] FastAPI 启动时 migration 自动添加 meetings 表新字段（幂等，捕获 `OperationalError`）
- [ ] FunASR 侧：`speaker_embeddings` 可读可写（Python `insert_speaker` / `list_speakers` / `delete_speaker`）
- [ ] FastAPI 侧：`transcript_json` 可存储完整 segments 数组 (JSON 往返无丢失)
- [ ] Rust sidecar：录音开始时创建，崩溃时标记 partial，恢复后删除
- [ ] Rust 在 `start_recording` 时将用户输入的 k 值写入 sidecar + 通过 API 写入 FastAPI `expected_speakers`
- [ ] 两个 SQLite 文件互不访问（Rust 不直连任何 SQLite，Python 不跨 DB 读写）

## grill-with-docs 确认 (2026-05-20)

- **speaker_embeddings 表归属**: FunASR 侧 `services/funasr/data.db`，FastAPI 侧已有的建表代码需删除（`backend/app/db.py:160-168`是历史错误）
- **meeting 创建流程**: 前端 `POST :5167/meetings` → 获得 meeting_id → 传给 Rust `start_recording`。会议开始前 Rust 侧不需要 meeting_id（由前端先行创建）
- **旧表处理**: `transcripts`/`transcript_chunks`/`summary_processes` 三张表在 Issue 06 第二阶段（删旧端点）时一并删除，不保留 fallback

## Blocked by

None — 可立即开始（基础设施，无上游依赖）
