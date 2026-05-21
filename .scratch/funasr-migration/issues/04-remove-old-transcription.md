# 04 — 删除 Whisper/Parakeet 旧代码

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

删除所有旧的 Whisper 和 Parakeet 转录相关代码，清理依赖，确保 `cargo build` 通过且转录功能不受影响。

端到端行为：`cargo build` 成功，录音 → 转录继续正常工作（由 #02 的新 FunASR 链路驱动）。旧 Rust 转录引擎不再编译，旧 Python Whisper 服务不再被引用。

**大工程，拆为 6 个独立 commit，每个 `cargo build` 可通过。利于 git bisect 定位，避免一次删除跨五六个模块导致难以 review。**

### Commit 04-1 — 删 `audio/transcription/` 模块

删除文件：
- `frontend/src-tauri/src/audio/transcription/whisper_provider.rs`
- `frontend/src-tauri/src/audio/transcription/parakeet_provider.rs`
- `frontend/src-tauri/src/audio/transcription/mod.rs`
- 如有其他 transcription/ 下文件一并删除

更新文件：
- `frontend/src-tauri/src/audio/mod.rs` — 移除 `pub mod transcription` 及所有 transcription 相关 `pub use`

验证：`cargo build` 通过（transcription 模块可能被 recording_commands 引用——这些引用在 02-3 已替换为 funasr task，应无编译错误）

### Commit 04-2 — 删 `whisper_engine/` 整个目录

删除文件：
- `frontend/src-tauri/src/whisper_engine/` 整个目录（含 commands、parallel_commands 等）

验证：`cargo build` 通过（whisper_engine 被 lib.rs 命令注册引用——先删目录，编译会因 lib.rs 引用报错，然后修复 lib.rs。**也可反过来：先修 lib.rs 再删目录——选任一，只要一次 commit 只处理一个模块**）

### Commit 04-3 — 删 `parakeet_engine/` 整个目录

删除文件：
- `frontend/src-tauri/src/parakeet_engine/` 整个目录

验证：`cargo build` 通过

### Commit 04-4 — 删 `audio/stt.rs` + `ContinuousVadProcessor` 残留 + 清理 `audio/mod.rs`

删除文件：
- `frontend/src-tauri/src/audio/stt.rs`

清理 `audio/mod.rs`：
- 移除 `pub use` 中所有已删除类型的导出
- 残留在 `ContiousVadProcessor` 的任何 import 或引用

注意：**不要删 `audio_processing::resample_audio()`**——FunASR 路径依赖它做 48k→16k 重采样。

### Commit 04-5 — `Cargo.toml` 删依赖

- 移除 whisper-rs、parakeet 等旧 transcription 相关 cargo 依赖
- 如有 feature flags 仅用于旧引擎，一并移除

验证：`cargo build` 通过（如果有未被 #02 引用的极隐式依赖可能刚删完报错，修复即可）

### Commit 04-6 — `lib.rs` 删旧 command 注册

删除 `tauri::generate_handler![]` 中以下命令的注册（约 30+ 条）：
- 所有 `whisper_engine::commands::*`
- 所有 `parakeet_engine::commands::*`
- ~~所有 `whisper_engine::parallel_commands::*`~~ (已在 04-2 随模块删除)
- 如有 transcription 相关其他命令

清理 `setup` hook 中的旧引擎初始化（`whisper_init`、`parakeet_init` 等 spawn 调用）

验证：`cargo build` 成功，`cargo check` 无 warning（未使用的导入等）

### 跨 commit 约束

每个 commit 必须满足：
- `cargo build` 成功，无编译错误
- 如果 `cargo build` 失败——必须修完再提交，否则 `git bisect` 会被卡住
- 每个 commit 后 git log 清晰——出问题时 `git bisect` 能定位到具体删除模块

## Acceptance criteria

- [ ] `cargo build` 成功，无编译错误（6 个 commit 各自通过）
- [ ] 无 Rust 编译警告（未使用的导入等）
- [ ] 启动 Tauri 应用，录音功能正常
- [ ] 转录通过 FunASR 链路工作（#02 的端到端功能不受影响）
- [ ] 无运行时 panic 或错误日志
- [ ] `audio_processing::resample_audio()` 未被误删

## Blocked by

#02（FunASR 转录链路）— 必须在新的转录链路跑通后才能删除旧代码
