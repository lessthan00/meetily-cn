# Issue B — Rust TDD + 管线重构 + 删旧代码

## 开始前必读（按顺序）

1. `.scratch/funasr-migration/issues/02-funasr-transcription-pipeline.md`
2. `.scratch/funasr-migration/issues/04-remove-old-transcription.md`
3. `docs/adr/003-voiceprint-speaker-diarization.md`
4. `CLAUDE.md`
5. `CONTEXT.md`
6. `.scratch/funasr-migration/tdd/README.md`

## 任务

分三个阶段，在同一个 agent 窗口内顺序执行：

1. **TDD 纯函数** — 实现 `.scratch/funasr-migration/tdd/` 中所有标注为 Rust 的 TDD 子任务，每个走 RED → GREEN → REFACTOR，每个循环一个 commit
2. **管线重构** — 删 VAD + 接线 FunASR（与阶段 1 的 ChunkAccumulator 衔接），1 个 commit。注意：删 VAD 和接线 FunASR 合并为原子 commit，避免中间态 transcription 不工作
3. **删除旧代码** — 按 issue 04 的删除清单逐步清理，每个 commit `cargo build` 必须通过

## 前提

- FunASR 服务已在 :8178 运行（至少 `/health` 返回 ok）
- 工作目录：`frontend/`

## 约束

- 不能删除 `audio_processing::resample_audio()`（管线重构依赖它）
- 不能修改 FunASR Python 服务代码
- 不能修改前端 React 代码
- 阶段间 `cargo build` 必须通过

## 完成标准

- 阶段 1：`cargo test` 全部通过
- 阶段 2：录音 → 前端出中文转录文本（E2E 通）
- 阶段 3：`cargo build` 通过，旧转录代码（whisper_engine/、parakeet_engine/、audio/transcription/、audio/stt.rs）全部删除
- 涉及的 issue 文件 acceptance criteria 全部打勾
