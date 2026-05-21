# FastAPI 摘要服务契约 (:5167)

**唯一 meetings 数据归属**。Rust 不直连此 SQLite，所有读写通过此 API。

---

## POST /meetings

**用途**：创建会议行。前端在录音开始前调用，获得 meeting_id 后传给 Rust `start_recording`。

**请求** (JSON body)：

```json
{"meeting_name": "团队站会"}
```

**响应**：

```json
{
  "meeting_id": "...",
  "meeting_name": "团队站会",
  "created_at": "2026-05-20T10:00:00"
}
```

---

## GET /meetings/{id}

**用途**：获取会议详情 (含 transcript_json segments)。前端详情页调用。

**响应**：

```json
{
  "meeting_id": "...",
  "meeting_name": "团队站会",
  "transcription_status": "complete",
  "expected_speakers": 3,
  "transcript_json": [
    {"start": 0.0, "end": 1.2, "text": "...", "speaker": "张甜甜"}
  ],
  "summary_json": "# 会议纪要\n...",
  "created_at": "..."
}
```

`transcript_json` 无数据时返回 `[]`（不 fallback 到旧表）。

---

## POST /save-transcript

**用途**：保存最终 segments (含真实姓名)。Rust 在命名完成后调用。

**请求** (JSON body)：

```json
{
  "meeting_id": "...",
  "segments": [
    {"start": 0.0, "end": 1.2, "text": "...", "speaker": "张甜甜"},
    {"start": 1.5, "end": 2.0, "text": "...", "speaker": "李四"}
  ]
}
```

**响应**：

```json
{"status": "ok"}
```

**容灾**：Rust 调此端点失败 → segments 写入 sidecar `named_segments` 字段，启动时扫描重试。

---

## POST /process-transcript

**用途**：生成摘要。从 `meetings.transcript_json` 列读 segments → Ollama qwen3:8b → 写入 `summary_json`。用户手动触发。

**请求** (JSON body)：

```json
{"meeting_id": "..."}
```

**响应**：

```json
{"status": "ok", "summary": "# 会议纪要\n..."}
```

摘要失败降级：返回 `{"status": "error", "message": "摘要生成失败，请重试"}`。

---

## POST /generate-title

**用途**：生成会议标题 (≤10 字)。会议结束后自动调用 (轻量，~1s)。

**请求** (JSON body)：

```json
{"meeting_id": "..."}
```

**响应**：

```json
{"status": "ok", "title": "Q2 产品路线图讨论"}
```

---

## GET /export/{meeting_id}?format=md

**用途**：Markdown 导出。读 `transcript_json` + `summary_json`，输出固定格式。

**响应** (Content-Type: text/markdown)：

```markdown
# 会议记录 — 2026-05-20 10:00

## AI 摘要
...摘要文本...

## 完整转录
**张甜甜** (00:00:00): 我认为这个方案可行。
**李四** (00:00:05): 但是有一些问题需要讨论。
```

---

## meetings 表结构

迁移后 (旧列 + 新增 5 列)：

| 列 | 类型 | 默认值 |
|---|---|---|
| (原有列) | — | — |
| `transcription_status` | TEXT | `'recording'` |
| `audio_file_path` | TEXT | NULL |
| `expected_speakers` | INTEGER | 3 |
| `transcript_json` | TEXT (JSON) | NULL |
| `summary_json` | TEXT (Markdown) | NULL |

旧表 (`transcripts`/`transcript_chunks`/`summary_processes`/`settings`/`transcript_settings`/`speaker_embeddings`) 在 Issue 06 第二阶段删除。
