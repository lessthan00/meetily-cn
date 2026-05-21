# 03 — 声纹注册 + 实时说话人标注 + 会后命名

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

实现声纹注册、会话管理、实时说话人标注和会后命名流程：用户注册声纹后，会议中已注册的说话人标注真实姓名，未注册的自动聚类编号。停止录音后通过 `/session/end` 获取聚类摘要，前端弹出命名界面，命名完成后才保存到 DB。

端到端行为：

- **注册**：用户打开声纹注册页 → 看到固定朗读文本 → 点击录音 → 朗读文本 → `POST /enroll` → 服务端校验音频（时长 ≥ 3s）→ CAM++ 提取 256 维声纹向量 → 存入 SQLite `speaker_embeddings` 表 → 前端显示「张甜甜，注册成功」

- **会议开始**：前端弹出输入框 → 用户输入预期说话人数 k（默认 3）→ Rust 调用 `POST /session/start {meeting_id, k}` → FunASR 初始化 k-means 聚类 → 返回 session_id

- **标注**：会议中 Rust 每 2 秒发送 chunk（携带 session_id）→ FunASR `/process-audio?mode=realtime` 内：
  1. fsmn-vad 切割语音段
  2. SenseVoiceSmall 逐段转录
  3. CAM++ 逐段提取声纹 → 余弦相似度比对
  4. 已注册（≥ 0.7）→ 返回姓名
  5. 未注册但匹配已有聚类质心 → 返回「说话人 A/B/…」
  6. 未注册且与所有质心 < 0.5 → 自适应扩容 k+1，新建聚类
  7. 介于 0.5–0.7 → 归入最近聚类（容忍一次）
  → 前端显示「**张甜甜**: 我觉得…」或「**说话人 A**: 我有个建议…」

- **会后命名 (grill-with-docs 确认)**：采用 **B + 持久化** 方案——命名完成后才保存，保证 DB 里不出现临时标签。
  ```
  停止录音 → Rust 调用 POST /session/end {session_id}
    → FunASR 返回 {speakers: [{label, segment_count, total_duration}]}
    → FunASR 销毁该 session 的 k-means 内存状态
    → Rust 将自身累加的 segments 写入 sidecar (raw_segments 字段) — 防刷新丢失
    → Rust emit "session-ended" 给前端 (含聚类摘要)

  前端弹出「未注册说话人命名」界面 (数据来源: session-ended event)
    → 用户输入: 说话人 A → "李四", 说话人 B → "王五"
    → 已注册说话人 (如 "张甜甜") 已在会议中标注，不在命名列表中
    → 跳过某说话人 → 保留「说话人 A」标签 (用户可接受不命名)

  用户确认命名 →
    → 前端调 Rust command: apply_speaker_names(meeting_id, {映射表})
    → Rust 从 sidecar 读取 segments，替换 speaker 字段
    → Rust emit "speakers-renamed" → 前端重新渲染转录面板
    → Rust POST :5167/save-transcript {meeting_id, segments} (带最终姓名，不含临时标签)
    → 成功后删除 sidecar 中 raw_segments 字段 (或删整个 sidecar)

  每位命名后提示：「是否为「李四」注册声纹？」
    ├ 是 → 跳转到声纹注册页 → 朗读固定文本 → POST /enroll
    └ 否 → 仅保存显示名，本次会议使用该名称
  ```

  **恢复场景**：如果用户在命名 UI 弹窗前刷新了页面——sidecar 中已有 `raw_segments` 字段。Rust 启动时扫描到残留 segments → 重新 emit "session-ended" → 前端弹出命名 UI。无需离线重处理，聚类标签保留不变。

### 涉及的代码区域

- **Python** `services/funasr/`：
  - `POST /enroll` — 接收音频 + 姓名，仅校验音频质量 (时长 ≥ 3s、信噪比、采样率 16kHz)，不校验朗读文本是否匹配。提取 CAM++ embedding，存入 SQLite。朗读文本 "北风与太阳" 仅作前端 UX 引导
  - `POST /identify` — 独立识别端点（调试/管理用）
  - `GET /speakers` — 返回声纹库列表
  - `DELETE /speakers/{id}` — 删除声纹
  - `GET /speakers/export` — 返回明文 JSON，含所有声纹向量。v2 加 AES-256-GCM 加密
  - `POST /speakers/import` — multipart: `file` (明文 JSON) → 增量合并（同名覆盖）。v2 加解密+密码
  - `POST /session/start` — 初始化 k-means 会话 {meeting_id, k} → {session_id}
  - `POST /session/end` — 结束会话，返回聚类摘要，销毁内存状态
  - 修改 `/process-audio?mode=realtime` — 增加 `session_id` 参数、CAM++ 声纹比对、在线 k-means 聚类逻辑
- **Rust** `funasr_client.rs`：
  - `enroll(name, audio)` — HTTP 调用 `/enroll`
  - `list_speakers()` — 调用 `GET /speakers`
  - `delete_speaker(id)` — 调用 `DELETE /speakers/{id}`
  - `export_speakers()` — 调用 `GET /speakers/export`
  - `import_speakers(file)` — 调用 `POST /speakers/import` (multipart: file)
  - `start_session(meeting_id, k)` — 调用 `POST /session/start`
  - `end_session(session_id)` — 调用 `POST /session/end`
  - `process_audio(wav_bytes, session_id)` — 修改现有方法，增加 session_id 参数
- **Rust** `apply_speaker_names` command (新建，在 `recording_commands.rs`)：
  - 接收 `meeting_id` + `{临时标签 → 最终姓名}` 映射表
  - 遍历 `RecordingSaver` 中所有 segments，替换 speaker 字段
  - emit `speakers-renamed` event 给前端
- **React**：
  - 声纹注册页（固定朗读文本展示 + 录音 + 注册按钮）
  - 声纹管理页：「导出全部」和「导入」按钮，导入时显示隐私提示
  - 会议开始前 k 值输入框（默认 3）
  - 转录面板：按 segment 渲染，已注册显示姓名，未注册显示「说话人 A」等聚类标签
  - 会后命名 UI（从 session-ended event 拿聚类摘要 → 输入框 → 确认 → apply_speaker_names → save-transcript → 可选注册声纹）
  - **重要**：save-transcript 调用移到命名完成后，不在 stop_recording 时立即保存

### 声纹注册数据结构（来自 ADR-003）

```
注册请求: POST /enroll
  body: { name: "张甜甜", audio: <WAV blob> }

响应: { success: true, speaker_id: 1, name: "张甜甜" }
      或 { success: false, error: "语音太短，请录制至少 3 秒" }
```

### 会话管理数据结构

```
POST /session/start
  body: { meeting_id: "...", k: 3 }
  response: { session_id: "..." }

POST /process-audio?mode=realtime
  multipart: audio (file) + session_id (field)
  response: { text: "...", segments: [{ start, end, text, speaker }, ...] }

POST /session/end
  body: { session_id: "..." }
  response: {
    speakers: [
      { label: "张甜甜", segment_count: 15, total_duration: 55.0 },
      { label: "说话人 A", segment_count: 12, total_duration: 45.2 },
      { label: "说话人 B", segment_count: 8,  total_duration: 30.1 }
    ]
  }
```

### 说话人标注阈值

- 余弦相似度 ≥ **0.7** → 匹配为已注册说话人
- < 0.7 且与某个聚类质心最近 → 标注聚类标签「说话人 A」
- < 0.5 且与所有质心都不近 → 自适应扩容，新建聚类

**阈值存储**: FunASR 侧 `config` 表 (`key='similarity_threshold', value='0.7'`)，启动时读取生效。前端设置页 `POST /config` 更新，FunASR 提供 `GET /config` 读取。v1 仅启动时读取——在线修改需重启 FunASR。

## Acceptance criteria

- [ ] 声纹注册页可访问，固定朗读文本清晰可见
- [ ] 注册「张甜甜」后，`GET /speakers` 返回该条目
- [ ] 录音 < 3 秒 → 返回明确错误提示
- [ ] 会议开始前显示 k 值输入框，默认 3
- [ ] `POST /session/start` 初始化成功，返回 session_id
- [ ] 同一人再次录音标注，转录旁显示「张甜甜」（阈值 ≥ 0.7 匹配成功）
- [ ] 未注册说话人标注为「说话人 A」而非「未知说话人」
- [ ] 会议中途新说话人加入 → 自适应扩容 k+1 → 前端提示「检测到未预期的说话人」
- [ ] 删除声纹后，该人不再被匹配（归入聚类标签）
- [ ] 阈值通过前端设置页修改后，匹配行为相应变化
- [ ] `GET /speakers/export` 返回含所有声纹向量的 JSON
- [ ] `POST /speakers/import` 增量合并（同名覆盖，不同名新增）
- [ ] 导入时前端提示隐私警告：「声纹数据包含生物特征，请确认来源可信」
- [ ] 停止录音后 `/session/end` 返回正确的聚类摘要
- [ ] 前端弹出命名 UI（已注册说话人不在列表中）
- [ ] 命名完成后 apply_speaker_names 更新 segments → emit speakers-renamed
- [ ] save-transcript 在命名完成后调用，DB 中不含「说话人 A」临时标签
- [ ] 命名后可选注册声纹 → 「是」跳转注册页，「否」仅保存显示名
- [ ] 刷新页面丢失命名进度时，从 sidecar `raw_segments` 恢复，无需离线重处理

## grill-with-docs 确认 (2026-05-20)

- **前端 ↔ FunASR 路径**:
  - 直连: `GET /health`, `GET /speakers`, `GET /config`, `POST /config` (纯 K-V 更新，无副作用，不需经 Rust)
  - 经 Rust (有副作用，需 Rust 协调): `POST /enroll`, `DELETE /speakers/{id}`, `POST /speakers/export`, `POST /speakers/import`
  - 转录链路: Rust → :8178 (`POST /process-audio`, `POST /session/start`, `POST /session/end`)
- **会后命名全跳过**: 允许用户一个都不命名直接关弹窗。DB 中保留「说话人 A」临时标签。v1 支持从会议详情页重新命名：点击任意临时标签 → 输入真实姓名 → 调 `apply_speaker_names` → 更新 `transcript_json`。与会后弹窗走同一条 command
- **首次启动引导**: 会议开始前若声纹库为空 → 弹窗提示「你还没有注册过声纹，是否现在注册？」→「跳过」/「去注册」
- **阈值热更新**: `POST /config` 写入时同步更新 FunASR 内存变量，即时生效，无需重启。前端 toast：「阈值已更新，已立即生效」
- **speaker_hints (v2 延后)**: 崩溃后重聚类时无法精确恢复标签映射（无 embedding 缓存），v1 前端标注「说话人信息因服务中断而不完整」，用户通过详情页重命名手动修复

## Blocked by

None — 可立即开始（与 #02 并行，仅依赖 #01 数据库）
