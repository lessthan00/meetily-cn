# 05 — 容灾恢复（崩溃 → 保存 → 重处理）

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

FunASR 服务崩溃/断开时，Rust 端按错误类型分层处理，自动保存完整原始录音，通过断路器停止发送 chunk，定期健康探测等待恢复。

### 崩溃检测（断路器）

| 错误类型 | 判定 | 处理 |
|---|---|---|
| `ECONNREFUSED` | 进程已死 | 立即打开断路器 |
| `ECONNRESET` | 进程崩溃中 | 立即打开断路器 |
| `ETIMEDOUT` | 可能 GPU hang | 重试 1 次，仍超时则打开断路器 |
| HTTP 5xx | 服务活着但内部出错 | 连续 3 次后打开断路器 |

**超时配置**：连接超时 5s，请求超时 10s

**断路器打开后**：
- 停止发送后续 chunk
- 将已收集的完整 WAV 保存到磁盘
- `INSERT/UPDATE meeting SET transcription_status = 'partial', audio_file_path = '/path/to/file.wav'`
- 前端弹出横幅：「转录服务中断，音频已保存」

### 健康探测与恢复

- Rust 健康探测使用渐进式间隔：首分钟 2s 一次（模型下载期间快速反馈），稳定后 15s 一次
- 连续 2 次成功 → 认为服务已恢复
- `status: "loading"` 时前端展示模型下载进度条（含百分比 + 模型名）
- `status: "ok"` 时认为服务就绪
- `status: "error"` 时展示错误信息

### 涉及的代码区域

- **Rust** `funasr_client.rs`：
  - 按错误类型分层处理（区分 ECONNREFUSED / ECONNRESET / ETIMEDOUT / HTTP 5xx）
  - 断路器逻辑：累计失败计数 → 打开断路器 → 停止发送
  - `GET /health` 健康探测（15s 间隔，连续 2 次成功算恢复）
  - WAV 累积保存逻辑（录音过程中收集所有 chunk 到 buffer，断路器打开时 flush 到文件）
- **Rust** 录音逻辑（如 `recording_manager`）：
  - 录音开始时前端已通过 `POST /meetings` 创建 meeting 行，Rust 仅创建 JSON sidecar
  - 录音正常结束 → Rust 调 POST /save-transcript 保存 segments，前端调 `/meetings/{id}` 更新 `transcription_status = 'complete'`
  - 断路器打开 → Rust 更新 sidecar `status = 'partial'` + 调 FastAPI 更新 `transcription_status = 'partial'` + `audio_file_path`
- **Python** `services/funasr/`：
  - `GET /health` 端点。响应格式: `{"status": "loading", "model": "SenseVoiceSmall", "progress": 45.0}` (首次下载中) / `{"status": "ok"}` (就绪) / `{"status": "error", "message": "..."}` (下载/启动失败)
  - `POST /process-audio?mode=offline` 端点：接受完整 WAV，返回完整转录 + 说话人标注
- **React**：
  - 崩溃横幅组件（从 meeting 状态读取 partial）
  - 「重新处理」按钮 → 触发离线处理 Tauri command
  - 服务恢复提示 → 横幅文案切换

### 离线模式降级策略

- 录音中 Rust 侧持续跟踪 `last_transcribed_end` (最近一次成功返回的 segments 中最大 `end` 值)，每次 FunASR 响应后更新
- **断路器打开时**：当前已累积的 segments 冻结写入 sidecar (`raw_segments`)，连同 `last_transcribed_end` 一并持久化
- **离线重处理**：传 `resume_from` 参数到 `POST /process-audio?mode=offline {audio, k: N, resume_from: 12.5}`。FunASR 只处理 `resume_from` 之后的音频段，返回的 segments 时间戳从该偏移量开始
- **拼接**：Rust 将 sidecar 中冻结的 pre-crash segments 与离线返回的 post-crash segments 拼接，中间插入恢复边界线
- **聚类**：离线模式内部创建临时 `SpeakerClusterer`，增量处理 resume_from 后的语音段。pre-crash 的 k-means 状态已随崩溃销毁，因此 post-crash 部分未注册说话人可能出现标签偏移（「说话人 A」→「说话人 C」）
- **恢复边界**：前端在 pre-crash 和 post-crash 部分之间渲染 `⚠ 转录服务于 14:32 中断，以下内容在恢复后生成` 分隔线。已注册说话人的标签不受影响（基于余弦比对，稳定）
- **v1 限制**：跨中断的未注册说话人标签不保证一致。v2 可在 `/session/start` 时序列化质心向量到 FunASR 侧 SQLite，崩溃后恢复聚类状态
- **会话管理**：离线模式不需要 `/session/start` + `/session/end`。端点内部创建临时 `SpeakerClusterer`，跑完返回 `speakers` 摘要 + `segments` 后自动销毁。响应格式与在线 `/session/end` 对齐，前端命名 UI 同一套代码消费

## Acceptance criteria

- [ ] Kill FunASR → `ECONNREFUSED` → 断路器立刻打开，前端显示崩溃横幅
- [ ] 网络超时（如防火墙阻断）→ 重试 1 次 → 仍超时 → 断路器打开
- [ ] HTTP 500 → 连续 3 次 → 断路器打开（不是 1 次就误判）
- [ ] 断路器打开后不再发送后续 chunk
- [ ] 数据库 meeting 行 `transcription_status = 'partial'`，`audio_file_path` 非空
- [ ] 保存的 WAV 文件可播放，时长完整
- [ ] 崩溃前已收到的转录片段正常显示（不丢失）
- [ ] FunASR 重启后 → 健康探测连续 2 次成功 → 前端提示恢复
- [ ] 点击「重新处理」→ 离线处理成功 → `transcription_status = 'complete'`
- [ ] 重新处理后转录文本无重复/截断
- [ ] 正常录音不受影响（无 false positive 断路器触发）

## grill-with-docs 确认 (2026-05-20)

- **sidecar 通用兜底**: `~/.meetily/recovery/{meeting_id}.json` 不局限于崩溃恢复——任何"尚未安全落库"的数据都写入 sidecar（崩溃保存、save-transcript 失败、命名后摘要服务不可用等）。Rust 启动时扫描 sidecar 目录，有残留数据则提示恢复
- **录音开始前 vs 录音中途崩溃分离**: 录音前 FunASR 未就绪 → 弹「启动引擎」不走断路器。录音中途崩溃 → 断路器逻辑
- **Tauri 自身崩溃**: 录音中每 5 分钟增量写 segments 到 sidecar。Tauri 崩溃 → 重启后 sidecar 有 `raw_segments` 和 `audio_file_path` → 恢复进度：已有 segments 保留，未转录部分用 `resume_from` 离线处理。录音文件永不删，确保可恢复

## Blocked by

#02（FunASR 转录链路） + #04（旧代码删除后架构定型）
