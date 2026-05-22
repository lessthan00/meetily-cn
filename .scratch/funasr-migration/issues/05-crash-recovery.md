# 05 — 容灾恢复（崩溃 → 保存 → 重处理）

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

FunASR 服务崩溃/断开时，Rust 端按错误类型分层处理，自动保存完整原始录音，通过断路器停止发送 chunk，定期健康探测等待恢复。

### 崩溃检测（断路器）— 6 种错误类型差异化处理 (Q15)

| # | 错误类型 | 策略 | 理由 |
|---|---|---|---|
| 1 | ECONNREFUSED | 立即打开 | 进程已死，无需再试 |
| 2 | ECONNRESET | 立即打开 | 进程崩溃中 |
| 3 | ETIMEDOUT (> 5s) | 重试 1 次，仍失败则打开 | 瞬态网络可自愈 |
| 4 | HTTP 5xx | 连续 3 次后打开 | 服务活着但出错，容忍抖动 |
| 5 | 503 `{"status":"unhealthy"}` | 连续 3 次后打开 | 模型加载中，类似 5xx |
| 6 | 反序列化失败 | 立即打开 | HTTP 200 + JSON 损坏 = 半崩溃 |

**超时配置**：连接超时 5s，请求超时 10s

**断路器打开后**：
- 停止发送后续 chunk
- 已累积的 WAV 已由 chunk-by-chunk 写盘保护(录音格式)
- 断路器打开后更新 sidecar 状态，前端弹出横幅：「转录服务中断，音频已保存」

### 健康探测与恢复

- Rust 健康探测使用渐进式间隔：首分钟 2s 一次，稳定后 15s 一次
- `GET /health` 三态响应：`loading` (模型下载中，含 progress) → **不触发断路器**，前端展示进度条 / `ok` → 就绪 / `error` → 计为 on_failure
- 半开状态：连续 2 次探测成功 → 切换到关闭。半开时正常 chunk 请求继续丢弃
- 探测失败 (半开中)：回到首分钟 2s 间隔，从头走渐进式

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

### 续传与恢复 (Q10)

- **resume_from 机制**：使用 sidecar 最后一条 segment 的 `seq` (非时间偏移)。断路器打开时 sidecar 中已有所有已转录 segments。恢复时 `POST /process-audio?mode=realtime&session_id=X&resume_from=42`
- **seq 去重**：前端侧按 `meeting_id + seq` 去重
- **k-means 恢复**：crash 后会话状态丢失 → 重新聚类。sidecar 中旧 segments 的 speaker 字段保留。聚类从零重建，声纹匹配继续 (SQLite 持久化)
- **离线重处理**：离线模式不需要 `/session/start` + `/session/end`。端点内部创建临时 `SpeakerClusterer`，跑完返回 `speakers` 摘要 + `segments` 后自动销毁。响应格式与在线 `/session/end` 对齐
- **恢复边界**：前端在 pre-crash 和 post-crash 之间渲染分隔线。已注册说话人标签不受影响。v1 接受未注册说话人标签偏移

### save-transcript 失败容灾 (Q10 补充)

- Rust 调 `POST /save-transcript` 返回非 200 → segments 保留在 sidecar (named=true 的已确认行)
- Rust 启动时扫描 sidecar → 尝试 `POST /save-transcript` 重试 (最多 3 次，间隔 5s)
- 全部失败 → 保留 sidecar，前端横幅提示。数据不丢

## Acceptance criteria

- [ ] Kill FunASR → `ECONNREFUSED` → 断路器立刻打开，前端显示崩溃横幅
- [ ] ECONNRESET → 断路器立刻打开
- [ ] 网络超时 ETIMEDOUT → 重试 1 次 → 仍超时 → 断路器打开
- [ ] HTTP 500 → 连续 3 次 → 断路器打开（不是 1 次就误判）
- [ ] 503 unhealthy → 连续 3 次 → 断路器打开
- [ ] 反序列化失败 → 断路器立刻打开
- [ ] `/health` 返回 `loading` → 不触发断路器。前端展示模型下载进度条
- [ ] 断路器打开后不再发送后续 chunk
- [ ] 崩溃前已收到的转录片段正常显示 (sidecar JSONL 保护)
- [ ] 半开状态：连续 2 次健康探测成功 → 恢复关闭
- [ ] 恢复后 segments 按 seq 去重，无重复/截断
- [ ] save-transcript 失败 → segments 保留 sidecar (named=true) → 启动时自动重试
- [ ] 正常录音不受影响（无 false positive 断路器触发）

## grill-with-docs 确认 (2026-05-20, 更新 2026-05-23)

- **sidecar 通用兜底**: `~/.meetily/recovery/{meeting_id}.jsonl` (JSONL 格式，每行一个 segment)。任何"尚未安全落库"的数据写入 sidecar。Rust 启动时扫描 sidecar 目录
- **录音开始前 vs 录音中途崩溃分离**: 录音前 FunASR 未就绪 → 弹「启动引擎」不走断路器。录音中途崩溃 → 断路器逻辑
- **Tauri 自身崩溃**: sidecar JSONL 每收到 FunASR 响应即追加写入 (O_APPEND)。Tauri 崩溃 → 重启后 sidecar 已有 segments → seq 去重恢复
- **Q15 6 种错误类型差异化**: ECONNREFUSED/ECONNRESET/反序列化失败 → 立即开。ETIMEDOUT → 重试 1 次。HTTP 5xx/503 unhealthy → 连续 3 次
- **Q10 seq-based resume**: 用 sidecar 最后一条 seq (非时间偏移)。更简单、无浮点精度问题
- **Q10 save-transcript 容灾**: save 失败 → sidecar 保留 → 启动时自动重试

## Blocked by

#02（FunASR 转录链路） + #04（旧代码删除后架构定型）
