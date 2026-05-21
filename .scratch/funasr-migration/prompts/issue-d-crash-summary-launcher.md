# Issue D — 容灾 + 摘要集成 + 启动脚本 + E2E

## 开始前必读（按顺序）

1. `.scratch/funasr-migration/issues/05-crash-recovery.md`
2. `.scratch/funasr-migration/issues/06-summary-service-and-export.md`
3. `.scratch/funasr-migration/issues/07-launcher-and-e2e.md`
4. `docs/adr/003-voiceprint-speaker-diarization.md`
5. `CLAUDE.md`
6. `CONTEXT.md`
7. `.scratch/funasr-migration/tdd/README.md`

## 任务

1. **断路器 TDD** — `.scratch/funasr-migration/tdd/05a-circuit-breaker.md`，RED → GREEN → REFACTOR
2. **容灾集成** — 断路器接入 FunASR 调用链路 + 健康探测 (渐进式间隔) + 崩溃保存 + offline 重处理
3. **摘要服务对齐** — `POST /save-transcript` (接收 segments → `transcript_json`) + `POST /process-transcript` (读 `transcript_json` → Ollama → `summary_json`) + `POST /generate-title` + `GET /export/{id}?format=md`
4. **一键启动脚本** — `start.sh` / `start.bat`（依赖检查 → 启动 :8178 + :5167 → 等待就绪 → 启动 Tauri）
5. **E2E 人工验证** — 按 issue 07 的验证清单逐项走

## 前提

- Issue A + B + C 已 merge 到 main
- FunASR (:8178) + FastAPI (:5167) 可启动

## 约束

- 不能修改 Issue A 的纯逻辑函数接口
- 不能修改 Issue C 创建的端点签名（仅新增端点或扩展响应格式）
- start.sh 不自动安装 CUDA 驱动、Docker、系统级音频驱动
- 摘要服务对齐分两个 commit：先加新端点，后删旧端点

## 完成标准

- `cargo test` (含断路器) 全部通过
- 录音中途 kill FunASR → 断路器触发 → 前端横幅 → 音频保存 → 重启 FunASR → 健康恢复 → 重新处理成功
- Ollama 摘要生成 + Markdown 导出格式正确
- 双击 start.sh → 3 个服务正常启动，Ctrl+C 无残留进程
- 涉及的 issue 文件 acceptance criteria 全部打勾
