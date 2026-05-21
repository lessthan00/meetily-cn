# FunASR 迁移 — PRD

基于 [ADR-003](../docs/adr/003-voiceprint-speaker-diarization.md) 的实施总览。

## 目标

用 FunASR (SenseVoiceSmall + CAM++) 替换现有 Whisper/Parakeet 转录方案，实现：
- 实时中文转录
- 声纹注册（CAM++ 256 维向量、固定朗读文本、SQLite 声纹库）
- 实时说话人标注（已注册 → 姓名；未注册 → 在线 k-means 聚类 →「说话人 A/B/C」）
- 会后命名 + 可选声纹注册流程
- 分层崩溃容灾恢复（断路器 + 健康探测）
- 摘要服务 + Markdown 导出对齐新数据格式

## 管线改造策略（grill-with-docs 确认）

**先删旧 VAD+Whisper 路径，再建 FunASR 路径。** git revert 即回归，无需并行比较。

旧：`AudioPipeline (混合 + VAD) → Whisper/Parakeet 推理`
新：`AudioPipeline (双路混音, 不做VAD) → 双路 48kHz 录音 (chunk-by-chunk 写磁盘) + 16kHz 混音 → ChunkAccumulator (2s) → FunasrClient → POST :8178`

**关键约束**：
- 录音 WAV 必须 chunk-by-chunk 追加写磁盘，禁止内存缓冲到结束才 flush
- 双路混音（麦克风 + 系统音频）必须保留，不可误删系统音频路径

## Agent 调度策略

**5 个 issue 窗口 (含 Issue-0 环境门禁)，顺序执行。TDD 按语言归堆，集成按模块归堆。** 每个窗口内不会出现「Python 写一半切 Rust 写 TDD」的情况。

## Issue-0 — 环境门禁

**语言**: Shell/Bash
**窗口**: Agent #0
**预估**: ~0.5h

内容见 [00-environment-validation.md](issues/00-environment-validation.md)。通过后输出「环境就绪」信号才能开始 Issue A。同时测量 `cargo test` 增量编译时间，若 > 60s 则配置 sccache。

## Issue A — Python TDD 纯逻辑 (含 Pilot Run)

**语言**: Python (pytest)
**窗口**: Agent #1
**预估**: ~6h (7 个 TDD 循环)

**Pilot Run**: 先跑 03b (k-means 聚类) — 最复杂的纯逻辑。如果在 1.5h 内通过，继续其余 6 个。如果超时或质量差 → 人工介入调整拆分。

逐个 commit，每个走 RED → GREEN → REFACTOR：

| # | Commit | TDD 文件 | 内容 |
|---|--------|---------|------|
| A1 | speaker_embeddings CRUD | [01a](tdd/01a-speaker-embedding-crud.md) | insert/list/delete/update 四个操作的 SQLite 封装 |
| A2 | 余弦相似度计算 | [03a](tdd/03a-cosine-similarity.md) | `cosine_similarity(a,b)` + `find_best_match(query, registry)` |
| A3 | k-means 质心分配 + 自适应扩容 | [03b](tdd/03b-kmeans-clustering.md) | `SpeakerClusterer` 类——整个项目最复杂的纯逻辑 |
| A4 | 音频时长/质量校验 | [03c](tdd/03c-audio-validation.md) | 校验 WAV 时长 ≥ 3s、信噪比、采样率 |
| A5 | 声纹 JSON 导入/导出 | [03d](tdd/03d-speaker-import-export.md) | 增量合并、同名覆盖、version 字段 |
| A6 | Markdown 导出格式 | [06a](tdd/06a-markdown-export-format.md) | `**姓名** (HH:MM:SS): 内容` 格式输出 |
| A7 | 摘要 prompt 构造 | [06b](tdd/06b-summary-prompt-construction.md) | 带 speaker 字段的 segments → Ollama prompt |

**约束**：7 个 TDD 循环互不依赖（CRUD 依赖 DB 表但可以先 mock），可按任意顺序执行。如果在同一个 agent 窗口内按 ID 顺序跑效果最好。

## Issue B — Rust TDD + 管线重构 + 删旧代码

**语言**: Rust (cargo test)
**窗口**: Agent #2 (基于 Issue A merge 后的 main)
**预估**: ~5h (3 TDD + 2 集成 + 6 删除)

三个阶段，在同一个 agent 窗口内连续执行：

### 阶段 1：TDD 纯函数 (3 commits)

每个走 RED → GREEN → REFACTOR，`cargo test` 各自独立：

| # | Commit | TDD 文件 | 内容 |
|---|--------|---------|------|
| B1 | ChunkAccumulator | [02c](tdd/02c-wav-chunk-accumulator.md) | 收 16kHz f32 → 满 32000 (2s) → 吐 chunk → 余数滚存 |
| B2 | segments JSON 反序列化 | [02a](tdd/02a-segment-deserialization.md) | `TranscriptionResult` 的 serde 正确性 |
| B3 | WAV header 构造 | [02b](tdd/02b-wav-header-construction.md) | f32 PCM → 16kHz mono WAV bytes |

### 阶段 2：管线重构 (1 commit)

> **提醒**: 删 VAD 和接线 FunASR 合并为 1 个 commit，避免中间态 transcription 不工作污染 bisect。

无 TDD——纯结构变更。commit 后 `cargo build` 必须通过：

| # | Commit | 内容 |
|---|--------|------|
| B4 | 改管线删 VAD + 接线 FunASR | `AudioPipeline` 内删 `ContinuousVadProcessor` 换 `ChunkAccumulator`，48k→16k 用 `audio_processing::resample_audio()`，`transcription_sender` 语义变为 2s raw chunk。新建 funasr transcription task 替代 `transcription::start_transcription_task()`：消费 2s chunk → B3 WAV → `FunasrClient::transcribe()` → B2 反序列化 → emit `transcript-update`。E2E 验证：录音 → 前端出中文 |

### 阶段 3：删除旧代码 (6 commits, 来自原 #04)

每个 commit `cargo build` 必须通过。顺序严格递增：

| # | Commit | 内容 |
|---|--------|------|
| B5 | 删 `audio/transcription/` | whisper_provider, parakeet_provider, mod.rs + 清理 `audio/mod.rs` |
| B6 | 删 `whisper_engine/` | 整个目录 + 修复 `lib.rs` 引用 |
| B7 | 删 `parakeet_engine/` | 整个目录 + 修复 `lib.rs` 引用 |
| B8 | 删 `audio/stt.rs` + VAD 残留 | 清理 `audio/mod.rs` 导出。**不得删 `audio_processing::resample_audio()`** |
| B9 | `Cargo.toml` 删依赖 | whisper-rs, parakeet 等旧依赖 |
| B10 | `lib.rs` 删旧命令注册 | 移除 whisper_engine/parakeet_engine 的 command 注册 + setup 中的 init spawn |

**约束**：B1–B3 可任意顺序（互不依赖），但 B4→B5→B6→…→B10 严格顺序。

## Issue C — FunASR 服务 + 声纹集成 + 前端

**语言**: Python (FastAPI) + Rust + React
**窗口**: Agent #3 (基于 Issue B merge 后的 main)
**预估**: ~4h

无 TDD（纯逻辑已在 Issue A 做完）。接线 + 集成 + UI：

| # | Commit | 内容 |
|---|--------|------|
| C1 | DB 迁移 + /enroll 端点 | **FunASR 侧**: 创建 `speaker_embeddings` 表 (幂等) + `POST /enroll` (接收 name + audio → 校验时长 → CAM++ 提取 embedding → 入库)。**FastAPI 侧**: meetings 表 migration 加 `transcription_status`/`audio_file_path`/`expected_speakers`/`transcript_json`/`summary_json` 列 (幂等)。Rust 不做自己的 SQLite——崩溃恢复状态用 JSON sidecar 文件 |
| C2 | /speakers CRUD + 导出/导入 | `GET /speakers`、`DELETE /speakers/{id}`、`GET /speakers/export`、`POST /speakers/import`。用 Issue A 的 A1/A5 函数 |
| C3 | CAM++ 集成到 /process-audio | 在 `/process-audio?mode=realtime` 中增加：fsmn-vad 切割 → SenseVoiceSmall → CAM++ 声纹 → 余弦比对 (A2) → k-means 聚类 (A3) → speaker 字段填充 |
| C3.5 | 会话管理端点 | `POST /session/start` (创建 session + 初始化 k-means, 返回 session_id)、`POST /session/end` (销毁 session + 返回聚类摘要)。后台 asyncio task 每 60s 扫描，`last_access` 超 4h 自动过期（会议常超 2h）。修改 `/process-audio` 接收 `session_id` 参数。~80 行 |
| C4 | Rust funasr_client 扩展方法 | `enroll()`、`list_speakers()`、`delete_speaker()`、`export_speakers()`、`import_speakers()`、`start_session()`、`end_session()`、传递 session_id |
| C5 | 前端 UI | 声纹注册页 (朗读固定文本 + 注册按钮)、声纹管理页 (导出/导入)、会议前 k 值输入框 (默认 4)、转录面板 speaker 渲染、会后命名提示 |

## Issue D — 容灾 + 摘要集成 + 启动脚本 + E2E

**语言**: Rust + Python + Shell
**窗口**: Agent #4 (基于 Issue C merge 后的 main)
**预估**: ~4h

1 TDD + 集成 + HITL：

| # | Commit | 内容 |
|---|--------|------|
| D1 | 断路器状态机 (TDD) | [05a](tdd/05a-circuit-breaker.md) — RED→GREEN→REFACTOR，`cargo test` |
| D2 | 容灾集成 | FunASR 崩溃检测 (ECONNREFUSED/ECONNRESET/ETIMEDOUT/HTTP 5xx) + 断路器打开 + 音频保存 + 健康探测 (渐进式: 首分钟 2s 间隔 → 稳定后 15s, 连续 2 次成功) + 模型下载进度展示 (前端「正在下载模型 45%」) + offline 重处理。session 生命周期集成测试 (start→process→end 链路，验证 session_id 不丢失不断连)。前端崩溃横幅 + 重新处理按钮 |
| D3 | 摘要服务对齐 | `POST /save-transcript` (Rust 传最终 segments → meetings.transcript_json) + `POST /process-transcript` (读 transcript_json → 摘要 prompt A7 → 写 summary_json) + `POST /generate-title` + 前端点击生成摘要 |
| D4 | Markdown 导出 | `GET /export/{meeting_id}?format=md` 输出格式 (A6)。读 `transcript_json` + `summary_json` → 转录 + 纪要合并 |
| D5 | 一键启动脚本 + E2E | `start.bat`/`start.sh` + 完整人工验证流程 (注册 2 人 → 3 人开会 → 转录 → 聚类 → 命名 → 摘要 → 导出 → 崩溃恢复) |

## 依赖图

```
Issue-0 (环境门禁)
  └──→ Issue A (Python TDD)
          └──→ Issue B (Rust TDD + 管线 + 删旧)
                  └──→ Issue C (声纹集成 + 前端)
                          └──→ Issue D (容灾 + 摘要 + 启动)
```

每个 Issue merge 到 main 后，下一个 Issue 基于 main 开新窗口。单线程 DeepSeek API，一次跑一个。

## 粒度验证 (Pilot Run)

在正式启动所有 TDD 之前，先拿最复杂的 TDD 子任务跑一轮完整循环作为 pilot：

- **标的**: 03b (k-means 聚类 — 整个项目最复杂的纯逻辑)
- **验证目标**:
  1. 实际 RED → GREEN → REFACTOR 耗时
  2. token 消耗量 (agent 上下文压力)
  3. 测试质量 (agent 隔离是否有效)
  4. 实现质量 (是否通过了真实测试而非走过场)
- **决策点**: 如果 03b 在 1.5h 内完成且质量合格 → 其余估时可信，继续。如果超时或质量差 → 调整估时，拆分过大的子任务

## 集成门禁

- **B4 (接线 commit)** 是第一个集成点，merge 后必须跑 E2E 冒烟：录音 10s → 前端出中文文本。不通过则 B1-B3 不算完成，不开 Issue C
- **C5 (前端 UI) 完成后**: 跑完整用户路径：注册 → 开 k 值 → 录音 → 看说话人标注

## TDD 子任务

总计 [12 个 TDD 子任务](./tdd/README.md)：

| Issue | TDD 数量 | 文件 |
|-------|---------|------|
| A | 7 | 01a, 03a, 03b, 03c, 03d, 06a, 06b |
| B | 3 | 02a, 02b, 02c |
| C | 0 | (纯集成) |
| D | 1 | 05a |

每个子任务硬性时限 1.5h，超时 → agent 输出失败报告，人工介入。严格 RED → GREEN → REFACTOR。
