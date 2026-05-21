# 02 — FunASR 转录链路 + SenseVoiceSmall

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

搭建完整的 FunASR 实时转录链路：Rust 端通过 HTTP 调用 FunASR 服务，获得真实中文转录文本并在前端展示。

端到端行为：用户点击开始录音 → Rust 采集 16kHz 音频 → 每 2 秒将 chunk 发送到 `POST /process-audio?mode=realtime` → Python FunASR 使用 fsmn-vad 切割语音段 → SenseVoiceSmall 逐段转录为中文文本 → 返回 JSON（含 segments 数组，每个 segment 带起止时间）→ Rust 通过 Tauri event 推送到前端 → React 转录面板实时显示文本。

### 关键设计决策（来自 ADR-003）

- **VAD 由 FunASR 独做**：Rust 不做 VAD，固定 2s chunk 发送原始音频
- **多说话人支持**：fsmn-vad 将 2s chunk 切割为语音段后逐段转录，一个 chunk 可返回多个 segment，每个 segment 可标注不同说话人（#03 接入声纹后生效，本 issue 中 speaker 字段暂为 null）
- **响应格式**：`segments` 数组，非单一片段
- **先删旧 VAD 路径再建新路径**：git revert 即回归，无需并行比较

### 管线改造策略（grill-with-docs 确认）

**数据流现状**：
```
麦克风/系统音频
  → AudioPipeline (混合 + ContinuousVadProcessor 做 VAD) 
  → transcription_sender 发送语音段 (16kHz)
  → transcription_receiver → transcription::start_transcription_task()
  → Whisper/Parakeet 推理 → emit "transcript-update"
```

**目标数据流**：
```
麦克风/系统音频
  → AudioPipeline (混合, 不做 VAD) 
  → 48k→16k 重采样 → 2s 固定 chunk (ChunkAccumulator)
  → transcription_sender → funasr_transcription_task
  → FunasrClient::transcribe() → POST :8178 → emit "transcript-update"
```

### 拆为 3 个独立 commit（每个 `cargo build` 可通过）

**Commit 02-1 — ChunkAccumulator (TDD: 02c)**
- 新建 `audio/chunk_accumulator.rs`
- 纯逻辑模块：收 16kHz f32 → 满 32000 采样 (2s) → 吐出 Vec<f32> → 余量滚入下一轮
- 不接线，仅编译通过
- 测试覆盖：空输入、正好 32000、64000、50000（余数滚存）、48k→16k 重采样后接入

> **提醒**: 02-2 和 02-3 合并为 1 个 commit——删 VAD + 接线 FunASR 原子完成，避免中间态 transcription 不工作污染 bisect。

**Commit 02-2 — 改管线删 VAD + 接线 FunASR**
- `AudioPipeline` 内部：删 `ContinuousVadProcessor`，换 `ChunkAccumulator`
- 分流路径：`mixed_clean` (48kHz) → 48k→16k 重采样 (用 `audio_processing::resample_audio()`) → `ChunkAccumulator` → 满 2s 时发到 `transcription_sender`
- 录音路径不受影响（`recording_sender_for_mixed` 保持原样）
- `transcription_sender` channel 语义从「VAD 语音段」变为「2s raw chunk」
- 注意：别删 `audio_processing::resample_audio()`，ChunkAccumulator 依赖它
- 新建 `start_funasr_transcription_task`：消费 `transcription_receiver` 中的 2s chunk → 调用 02b 构造 WAV → `FunasrClient::transcribe()` → 调用 02a 反序列化 segments → emit `transcript-update`
- `recording_commands::start_recording_with_devices_and_meeting()` 中替换 `transcription::start_transcription_task` 为新的 funasr task
- E2E 跑通：录音 → 前端出中文文本
- 注意：`funasr_client.rs` 需增加 `Clone` derive 以在 tokio task 中共享

### 涉及的代码区域

- **Rust** `frontend/src-tauri/src/audio/chunk_accumulator.rs` — 新建（02-1，TDD 02c）
- **Rust** `frontend/src-tauri/src/audio/pipeline.rs` — 删 VAD，加 ChunkAccumulator（02-2）
- **Rust** `frontend/src-tauri/src/funasr_client.rs` — HTTP 客户端（已骨架，02-3 接线）
- **Rust** `frontend/src-tauri/src/audio/recording_commands.rs` — 替换 transcription task（02-3）
- **React** — 确保现有的 transcript-update 事件监听器能正确渲染 segments 数组

### API 规范（来自 ADR-003）

**请求**：`POST /process-audio?mode=realtime`
- Content-Type: `multipart/form-data`
- Body: `audio` 字段，16kHz mono PCM WAV

**响应**（JSON）：
```json
{
  "text": "转录文本内容（所有 segment 拼接）",
  "segments": [
    {"start": 0.0, "end": 1.2, "text": "我觉得这个方案可行", "speaker": null},
    {"start": 1.5, "end": 2.0, "text": "嗯我也同意", "speaker": null}
  ]
}
```

> 注：speaker 字段在 segment 层级（非顶层）。本 issue 中 speaker 暂为 null，#03 接入声纹后逐段填充姓名或聚类标签。`segments` 数组是响应主体——前端应按 segment 逐条渲染，而非仅展示 `text` 字段。

## Acceptance criteria

- [ ] FunASR 服务在 :8178 启动成功，使用 **FastAPI** 框架（与 :5167 一致，AI 维护模式复用）
- [ ] `funasr_client.rs` 实现完整的胖客户端：一个 struct 封装 :8178 全部端点（transcribe/start_session/end_session/enroll/speakers CRUD/health）。~180 行，AI 一眼看到全部 API 表面
- [ ] 02-1: `ChunkAccumulator` 测试通过（空/正好/多 chunk/余数滚存）
- [ ] 02-2: `cargo build` 通过，无 Compilation 错误
- [ ] 02-3: Rust `funasr_client.rs` 能成功调用端点，解析 segments 数组
- [ ] 02-3: 录音 2 秒后前端转录面板出现中文文本（非乱码）
- [ ] 持续录音 30 秒后，多条转录片段依次出现，不丢帧
- [ ] FunASR 首次启动自动下载 SenseVoiceSmall 模型，第二次启动直接加载
- [ ] FunASR 服务不可用时，前端显示错误提示（而非静默丢弃）

## grill-with-docs 确认 (2026-05-20)

- **FunASR 未就绪处理**: 录音按钮按下时，Rust 调 `POST /session/start` → `ECONNREFUSED` → **不走断路器**，前端弹「语音引擎未启动，请运行 start.sh 或点击启动引擎」+ 「启动引擎」按钮（调 `start-funasr.sh`）。与录音中途崩溃（走断路器）是两条路径
- **健康检查路径**: 前端直连 `GET :8178/health`（2s 轮询），不经过 Rust
- **模型下载进度**: `/health` 返回 `{status: "loading", model: "...", progress: 45.0}` → 前端直接解析展示进度条。这条也适用于首次启动（用户双击 Tauri 而非 start.sh 时）
- **CORS**: FunASR 服务设 `allow_origins=["*"]`。纯本地部署，无安全风险。Tauri webview 的 origin 可能为 `tauri://localhost` 或自定义 scheme，`*` 统一覆盖
- **错误信封**: FunASR 所有端点统一返回 `{"status": "ok"|"error", ...}`。应用层错误 (status=error) 与传输层错误 (ECONNREFUSED 等) 是两个独立层次，各自处理

## Blocked by

None — 可立即开始（与 #03 声纹注册并行开发）
