# Issue C — FunASR + 声纹集成 + 前端

## 开始前必读（按顺序）

1. `.scratch/funasr-migration/issues/03-voiceprint-enrollment-and-identification.md`
2. `docs/adr/003-voiceprint-speaker-diarization.md`
3. `CLAUDE.md`
4. `CONTEXT.md`

## 任务

基于 Issue A 的纯逻辑模块和 Issue B 的管线，实现完整的 FunASR 服务端点、声纹注册/识别集成、会话管理和前端 UI。

**无 TDD** — 纯逻辑已在 Issue A 完成，本窗口只做接线和集成。

工作范围：Python (`services/funasr/` + `backend/app/db.py`) + Rust (`funasr_client.rs` + `recording_commands.rs`) + React 前端

## 前提

- Issue A + B 已 merge 到 main
- FunASR 依赖已安装（`pip install funasr`）

## 约束

- 不能修改 Issue A 产出的纯逻辑函数接口，只能 import 调用
- 不能在 Rust 侧创建 SQLite 数据库
- speaker 字段仅存在于 segment 层级，不在顶层

## 完成标准

- `POST /enroll` 可注册声纹，`GET /speakers` 返回列表
- `POST /session/start` → `/process-audio?mode=realtime` (携带 session_id) → `/session/end` 整条链路通
- 已注册说话人标注姓名，未注册标注「说话人 A/B/C」
- 会后命名 UI 弹出 → 用户命名 → apply_speaker_names → save-transcript（DB 无临时标签）
- 涉及的 issue 文件 acceptance criteria 全部打勾
