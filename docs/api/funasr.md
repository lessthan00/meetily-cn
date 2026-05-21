# FunASR API 契约 (:8178)

所有端点统一返回信封：`{"status": "ok"|"error", ...}`。应用层 error 与传输层 error (ECONNREFUSED 等) 分开处理。

CORS: `allow_origins=["*"]` (全本地部署)。

---

## GET /health

**用途**：健康检查 + 模型下载进度。前端直连，2s 轮询。

**响应**：

```json
// 就绪
{"status": "ok"}

// 模型下载中
{"status": "loading", "model": "SenseVoiceSmall", "progress": 45.0}

// 启动失败
{"status": "error", "message": "Model download failed: ..."}
```

---

## POST /process-audio

**用途**：转录 + 声纹标注。

### 实时模式

`POST /process-audio?mode=realtime`

**请求**：multipart/form-data
- `audio`: 16kHz mono PCM WAV (32-bit float)
- `session_id`: string (来自 /session/start)

**响应**：

```json
{
  "status": "ok",
  "text": "我觉得这个方案可行。嗯我也同意。",
  "segments": [
    {"start": 0.0, "end": 1.2, "text": "我觉得这个方案可行", "speaker": "张甜甜"},
    {"start": 1.5, "end": 2.0, "text": "嗯我也同意", "speaker": "说话人 A"}
  ]
}
```

speaker 字段类型：`string | null`。已注册说话人返回姓名，未注册返回聚类标签。

### 离线模式

`POST /process-audio?mode=offline`

**请求**：multipart/form-data
- `audio`: 完整 16kHz mono PCM WAV
- `k`: int (期望说话人数)
- `resume_from`: float | null (断点续传偏移秒数，崩溃恢复用)

**响应**：与实时模式格式相同。内部创建临时 SpeakerClusterer，返回 segments + speakers 摘要后销毁。

---

## POST /enroll

**用途**：声纹注册。经 Rust 调用。

**请求**：multipart/form-data
- `name`: string (说话人姓名)
- `audio`: 16kHz mono PCM WAV

**校验**：仅校验音频质量 — 时长 ≥ 3s、信噪比、采样率 16kHz。不校验文本是否匹配"北风与太阳"。

**响应**：

```json
// 成功
{"status": "ok", "speaker_id": 1, "name": "张甜甜"}

// 失败
{"status": "error", "message": "语音太短，请录制至少 3 秒"}
```

---

## POST /identify

**用途**：独立声纹识别 (调试/管理用)。经 Rust 调用。

**请求**：multipart/form-data
- `audio`: 16kHz mono PCM WAV

**响应**：

```json
{"status": "ok", "speaker": {"id": 1, "name": "张甜甜", "similarity": 0.85}}
// 或
{"status": "ok", "speaker": null}
```

---

## GET /speakers

**用途**：列出所有已注册说话人。前端直连。

**响应**：

```json
{
  "status": "ok",
  "speakers": [
    {"id": 1, "name": "张甜甜", "created_at": "2026-05-20T10:00:00"}
  ]
}
```

---

## DELETE /speakers/{id}

**用途**：删除声纹。经 Rust 调用。

**响应**：

```json
{"status": "ok"}
```

---

## GET /speakers/export

**用途**：导出明文 JSON 声纹库。经 Rust 调用。v2 加加密。

**响应**：

```json
{
  "status": "ok",
  "version": 1,
  "speakers": [
    {"id": 1, "name": "张甜甜", "embedding": [0.1, 0.2, ...], "created_at": "..."}
  ]
}
```

---

## POST /speakers/import

**用途**：导入明文 JSON 声纹库。增量合并 (同名覆盖)。经 Rust 调用。v2 加加密。

**请求**：multipart/form-data
- `file`: JSON 文件 (格式同 export 响应)

**响应**：

```json
{"status": "ok", "imported": 3, "updated": 1}
```

---

## POST /session/start

**用途**：创建说话人聚类会话。经 Rust 调用 (录音开始时)。

**请求** (JSON body)：

```json
{"meeting_id": "...", "k": 3}
```

**响应**：

```json
{"status": "ok", "session_id": "..."}
```

---

## POST /session/end

**用途**：销毁会话，返回聚类摘要。经 Rust 调用 (录音结束时)。

**请求** (JSON body)：

```json
{"session_id": "..."}
```

**响应**：

```json
{
  "status": "ok",
  "speakers": [
    {"label": "张甜甜", "segment_count": 15, "total_duration": 55.0},
    {"label": "说话人 A", "segment_count": 12, "total_duration": 45.2},
    {"label": "说话人 B", "segment_count": 8,  "total_duration": 30.1}
  ]
}
```

session 超时：4h (last_access > 4h → 自动过期销毁)。

---

## GET /config

**用途**：读取配置。前端直连。

**响应**：

```json
{"status": "ok", "similarity_threshold": 0.7}
```

---

## POST /config

**用途**：更新配置。前端直连。即时生效 (更新内存变量，无需重启)。

**请求** (JSON body)：

```json
{"similarity_threshold": 0.75}
```

**响应**：

```json
{"status": "ok"}
```
