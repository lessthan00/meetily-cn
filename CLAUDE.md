# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Meetily** is a privacy-first AI meeting assistant that captures, transcribes, and summarizes meetings entirely on local infrastructure. The project consists of two main components:

1. **Frontend**: Tauri-based desktop application (Rust + Next.js + TypeScript)
2. **Backend**: FastAPI server for meeting storage and LLM-based summarization (Python)

### Key Technology Stack
- **Desktop App**: Tauri 2.x (Rust) + Next.js 14 + React 18
- **Audio Processing**: Rust (cpal, whisper-rs, professional audio mixing)
- **Transcription**: Whisper.cpp (local, GPU-accelerated)
- **Backend API**: FastAPI + SQLite (aiosqlite)
- **LLM Integration**: Ollama (local), Claude, Groq, OpenRouter

## Essential Development Commands

### Frontend Development (Tauri Desktop App)

**Location**: `/frontend`

```bash
# macOS Development
./clean_run.sh              # Clean build and run with info logging
./clean_run.sh debug        # Run with debug logging
./clean_build.sh            # Production build

# Windows Development
clean_run_windows.bat       # Clean build and run
clean_build_windows.bat     # Production build

# Manual Commands
pnpm install                # Install dependencies
pnpm run dev                # Next.js dev server (port 3118)
pnpm run tauri:dev          # Full Tauri development mode
pnpm run tauri:build        # Production build

# GPU-Specific Builds (for testing acceleration)
pnpm run tauri:dev:metal    # macOS Metal GPU
pnpm run tauri:dev:cuda     # NVIDIA CUDA
pnpm run tauri:dev:vulkan   # AMD/Intel Vulkan
pnpm run tauri:dev:cpu      # CPU-only (no GPU)
```

### Backend Development (FastAPI Server)

**Location**: `/backend`

```bash
# macOS
./build_whisper.sh small              # Build Whisper with 'small' model
./clean_start_backend.sh              # Start FastAPI server (port 5167)

# Windows
build_whisper.cmd small               # Build Whisper with model
start_with_output.ps1                 # Interactive setup and start
clean_start_backend.cmd               # Start server

# Docker (Cross-Platform)
./run-docker.sh start --interactive   # Interactive setup (macOS/Linux)
.\run-docker.ps1 start -Interactive   # Interactive setup (Windows)
./run-docker.sh logs --service app    # View logs
```

**Available Whisper Models**: `tiny`, `tiny.en`, `base`, `base.en`, `small`, `small.en`, `medium`, `medium.en`, `large-v1`, `large-v2`, `large-v3`, `large-v3-turbo`

### Service Endpoints
- **Whisper Server**: http://localhost:8178
- **Backend API**: http://localhost:5167
- **Backend Docs**: http://localhost:5167/docs
- **Frontend Dev**: http://localhost:3118

## High-Level Architecture

### Three-Tier System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Frontend (Tauri Desktop App)                  │
│  ┌──────────────────┐  ┌─────────────────┐  ┌────────────────┐ │
│  │   Next.js UI     │  │  Rust Backend   │  │ Whisper Engine │ │
│  │  (React/TS)      │←→│  (Audio + IPC)  │←→│  (Local STT)   │ │
│  └──────────────────┘  └─────────────────┘  └────────────────┘ │
│         ↑ Tauri Events           ↑ Audio Pipeline               │
└─────────┼────────────────────────┼─────────────────────────────┘
          │ HTTP/WebSocket         │
          ↓                        │
┌─────────────────────────────────┼─────────────────────────────┐
│              Backend (FastAPI)  │                              │
│  ┌────────────┐  ┌─────────────┴──────┐  ┌────────────────┐  │
│  │   SQLite   │←→│  Meeting Manager   │←→│  LLM Provider  │  │
│  │ (Meetings) │  │  (CRUD + Summary)  │  │ (Ollama/etc.)  │  │
│  └────────────┘  └────────────────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Audio Processing Pipeline (Critical Understanding)

The audio system has **two parallel paths** with different purposes:

```
Raw Audio (Mic + System)
         ↓
┌────────────────────────────────────────────────────────────┐
│              Audio Pipeline Manager                         │
│  (frontend/src-tauri/src/audio/pipeline.rs)                │
└─────────────┬──────────────────────────┬───────────────────┘
              ↓                          ↓
    ┌─────────────────┐        ┌─────────────────────┐
    │ Recording Path  │        │ Transcription Path  │
    │ (Pre-mixed)     │        │ (VAD-filtered)      │
    └─────────────────┘        └─────────────────────┘
              ↓                          ↓
    RecordingSaver.save()      WhisperEngine.transcribe()
```

**Key Insight**: The pipeline performs **professional audio mixing** (RMS-based ducking, clipping prevention) for recording, while simultaneously applying **Voice Activity Detection (VAD)** to send only speech segments to Whisper for transcription.

### Audio Device Modularization (Recently Completed)

**Context**: The audio system was refactored from a monolithic 1028-line `core.rs` file into focused modules. See [AUDIO_MODULARIZATION_PLAN.md](AUDIO_MODULARIZATION_PLAN.md) for details.

```
audio/
├── devices/                    # Device discovery and configuration
│   ├── discovery.rs           # list_audio_devices, trigger_audio_permission
│   ├── microphone.rs          # default_input_device
│   ├── speakers.rs            # default_output_device
│   ├── configuration.rs       # AudioDevice types, parsing
│   └── platform/              # Platform-specific implementations
│       ├── windows.rs         # WASAPI logic (~200 lines)
│       ├── macos.rs           # ScreenCaptureKit logic
│       └── linux.rs           # ALSA/PulseAudio logic
├── capture/                   # Audio stream capture
│   ├── microphone.rs          # Microphone capture stream
│   ├── system.rs              # System audio capture stream
│   └── core_audio.rs          # macOS ScreenCaptureKit integration
├── pipeline.rs                # Audio mixing and VAD processing
├── recording_manager.rs       # High-level recording coordination
├── recording_commands.rs      # Tauri command interface
└── recording_saver.rs         # Audio file writing
```

**When working on audio features**:
- Device detection issues → `devices/discovery.rs` or `devices/platform/{windows,macos,linux}.rs`
- Microphone/speaker problems → `devices/microphone.rs` or `devices/speakers.rs`
- Audio capture issues → `capture/microphone.rs` or `capture/system.rs`
- Mixing/processing problems → `pipeline.rs`
- Recording workflow → `recording_manager.rs`

### Rust ↔ Frontend Communication (Tauri Architecture)

**Command Pattern** (Frontend → Rust):
```typescript
// Frontend: src/app/page.tsx
await invoke('start_recording', {
  mic_device_name: "Built-in Microphone",
  system_device_name: "BlackHole 2ch",
  meeting_name: "Team Standup"
});
```

```rust
// Rust: src/lib.rs
#[tauri::command]
async fn start_recording<R: Runtime>(
    app: AppHandle<R>,
    mic_device_name: Option<String>,
    system_device_name: Option<String>,
    meeting_name: Option<String>
) -> Result<(), String> {
    // Implementation delegates to audio::recording_commands
}
```

**Event Pattern** (Rust → Frontend):
```rust
// Rust: Emit transcript updates
app.emit("transcript-update", TranscriptUpdate {
    text: "Hello world".to_string(),
    timestamp: chrono::Utc::now(),
    // ...
})?;
```

```typescript
// Frontend: Listen for events
await listen<TranscriptUpdate>('transcript-update', (event) => {
  setTranscripts(prev => [...prev, event.payload]);
});
```

### Whisper Model Management

**Model Storage Locations**:
- **Development**: `frontend/models/` or `backend/whisper-server-package/models/`
- **Production (macOS)**: `~/Library/Application Support/Meetily/models/`
- **Production (Windows)**: `%APPDATA%\Meetily\models\`

**Model Loading** (frontend/src-tauri/src/whisper_engine/whisper_engine.rs):
```rust
pub async fn load_model(&self, model_name: &str) -> Result<()> {
    // Automatically detects GPU capabilities (Metal/CUDA/Vulkan)
    // Falls back to CPU if GPU unavailable
}
```

**GPU Acceleration**:
- **macOS**: Metal + CoreML (automatically enabled)
- **Windows/Linux**: CUDA (NVIDIA), Vulkan (AMD/Intel), or CPU
- Configure via Cargo features: `--features cuda`, `--features vulkan`

## Critical Development Patterns

### 1. Audio Buffer Management

**Ring Buffer Mixing** (pipeline.rs):
- Mic and system audio arrive asynchronously at different rates
- Ring buffer accumulates samples until both streams have aligned windows (50ms)
- Professional mixing applies RMS-based ducking to prevent system audio from drowning out microphone
- Uses `VecDeque` for efficient windowed processing

### 2. Thread Safety and Async Boundaries

**Recording State** (recording_state.rs):
```rust
pub struct RecordingState {
    is_recording: Arc<AtomicBool>,
    audio_sender: Arc<RwLock<Option<mpsc::UnboundedSender<AudioChunk>>>>,
    // ...
}
```

**Key Pattern**: Use `Arc<RwLock<T>>` for shared state across async tasks, `Arc<AtomicBool>` for simple flags.

### 3. Error Handling and Logging

**Performance-Aware Logging** (lib.rs):
```rust
#[cfg(debug_assertions)]
macro_rules! perf_debug {
    ($($arg:tt)*) => { log::debug!($($arg)*) };
}

#[cfg(not(debug_assertions))]
macro_rules! perf_debug {
    ($($arg:tt)*) => {};  // Zero overhead in release builds
}
```

**Usage**: Use `perf_debug!()` and `perf_trace!()` for hot-path logging that should be eliminated in production.

### 4. Frontend State Management

**Sidebar Context** (components/Sidebar/SidebarProvider.tsx):
- Global state for meetings list, current meeting, recording status
- Communicates with backend API (http://localhost:5167)
- Manages WebSocket connections for real-time updates

**Pattern**: Tauri commands update Rust state → Emit events → Frontend listeners update React state → Context propagates to components

## Common Development Tasks

### Adding a New Audio Device Platform

1. Create platform file: `audio/devices/platform/{platform_name}.rs`
2. Implement device enumeration for the platform
3. Add platform-specific configuration in `audio/devices/configuration.rs`
4. Update `audio/devices/platform/mod.rs` to export new platform functions
5. Test with `cargo check` and platform-specific device tests

### Adding a New Tauri Command

1. Define command in `src/lib.rs`:
   ```rust
   #[tauri::command]
   async fn my_command(arg: String) -> Result<String, String> { /* ... */ }
   ```
2. Register in `tauri::Builder`:
   ```rust
   .invoke_handler(tauri::generate_handler![
       start_recording,
       my_command,  // Add here
   ])
   ```
3. Call from frontend:
   ```typescript
   const result = await invoke<string>('my_command', { arg: 'value' });
   ```

### Modifying Audio Pipeline Behavior

**Location**: `frontend/src-tauri/src/audio/pipeline.rs`

Key components:
- `AudioMixerRingBuffer`: Manages mic + system audio synchronization
- `ProfessionalAudioMixer`: RMS-based ducking and mixing
- `AudioPipelineManager`: Orchestrates VAD, mixing, and distribution

**Testing Audio Changes**:
```bash
# Enable verbose audio logging
RUST_LOG=app_lib::audio=debug ./clean_run.sh

# Monitor audio metrics in real-time
# Check Developer Console in the app (Cmd+Shift+I on macOS)
```

### Backend API Development

**Adding New Endpoints** (backend/app/main.py):
```python
@app.post("/api/my-endpoint")
async def my_endpoint(request: MyRequest) -> MyResponse:
    # Use DatabaseManager for persistence
    db = DatabaseManager()
    result = await db.some_operation()
    return result
```

**Database Operations** (backend/app/db.py):
- All meeting data stored in SQLite
- Use `DatabaseManager` class for all DB operations
- Async operations with `aiosqlite`

## Testing and Debugging

### Frontend Debugging

**Enable Rust Logging**:
```bash
# macOS
RUST_LOG=debug ./clean_run.sh

# Windows (PowerShell)
$env:RUST_LOG="debug"; ./clean_run_windows.bat
```

**Developer Tools**:
- Open DevTools: `Cmd+Shift+I` (macOS) or `Ctrl+Shift+I` (Windows)
- Console Toggle: Built into app UI (console icon)
- View Rust logs: Check terminal output

### Backend Debugging

**View API Logs**:
```bash
# Backend logs show in terminal with detailed formatting:
# 2025-01-03 12:34:56 - INFO - [main.py:123 - endpoint_name()] - Message
```

**Test API Directly**:
- Swagger UI: http://localhost:5167/docs
- ReDoc: http://localhost:5167/redoc

### Audio Pipeline Debugging

**Key Metrics** (emitted by pipeline):
- Buffer sizes (mic/system)
- Mixing window count
- VAD detection rate
- Dropped chunk warnings

**Monitor via Developer Console**: The app includes real-time metrics display when recording.

## Platform-Specific Notes

### macOS
- **Audio Capture**: Uses ScreenCaptureKit for system audio (macOS 13+)
- **GPU**: Metal + CoreML automatically enabled
- **Permissions**: Requires microphone + screen recording permissions
- **System Audio**: Requires virtual audio device (BlackHole) for system capture

### Windows
- **Audio Capture**: Uses WASAPI (Windows Audio Session API)
- **GPU**: CUDA (NVIDIA) or Vulkan (AMD/Intel) via Cargo features
- **Build Tools**: Requires Visual Studio Build Tools with C++ workload
- **System Audio**: Uses WASAPI loopback for system capture

### Linux
- **Audio Capture**: ALSA/PulseAudio
- **GPU**: CUDA (NVIDIA) or Vulkan via Cargo features
- **Dependencies**: Requires cmake, llvm, libomp

## Performance Optimization Guidelines

### Audio Processing
- Use `perf_debug!()` / `perf_trace!()` for hot-path logging (zero cost in release)
- Batch audio metrics using `AudioMetricsBatcher` (pipeline.rs)
- Pre-allocate buffers with `AudioBufferPool` (buffer_pool.rs)
- VAD filtering reduces Whisper load by ~70% (only processes speech)

### Whisper Transcription
- **Model Selection**: Balance accuracy vs speed
  - Development: `base` or `small` (fast iteration)
  - Production: `medium` or `large-v3` (best quality)
- **GPU Acceleration**: 5-10x faster than CPU
- **Parallel Processing**: Available in `whisper_engine/parallel_processor.rs` for batch workloads

### Frontend Performance
- React state updates batched via Sidebar context
- Transcript rendering virtualized for large meetings
- Audio level monitoring throttled to 60fps

## Important Constraints and Gotchas

1. **Audio Chunk Size**: Pipeline expects consistent 48kHz sample rate. Resampling happens at capture time.

2. **Platform Audio Quirks**:
   - macOS: ScreenCaptureKit requires macOS 13+, needs screen recording permission
   - Windows: WASAPI exclusive mode can conflict with other apps
   - System audio requires virtual device (BlackHole on macOS, WASAPI loopback on Windows)

3. **Whisper Model Loading**: Models are loaded once and cached. Changing models requires app restart or manual unload/reload.

4. **Backend Dependency**: Frontend can run standalone (local Whisper), but meeting persistence and LLM features require backend running.

5. **CORS Configuration**: Backend allows all origins (`"*"`) for development. Restrict for production deployment.

6. **File Paths**: Use Tauri's path APIs (`downloadDir`, etc.) for cross-platform compatibility. Never hardcode paths.

7. **Audio Permissions**: Request permissions early. macOS requires both microphone AND screen recording for system audio.

## Repository-Specific Conventions

- **Logging Format**: Backend uses detailed formatting with filename:line:function
- **Error Handling**: Rust uses `anyhow::Result`, frontend uses try-catch with user-friendly messages
- **Naming**: Audio devices use "microphone" and "system" consistently (not "input"/"output")
- **Git Branches**:
  - `main`: Stable releases
  - `fix/*`: Bug fixes
  - `enhance/*`: Feature enhancements
  - Current: `fix/audio-mixing` (working on audio pipeline improvements)

## Key Files Reference

**Core Coordination**:
- [frontend/src-tauri/src/lib.rs](frontend/src-tauri/src/lib.rs) - Main Tauri entry point, command registration
- [frontend/src-tauri/src/audio/mod.rs](frontend/src-tauri/src/audio/mod.rs) - Audio module exports
- [backend/app/main.py](backend/app/main.py) - FastAPI application, API endpoints

**Audio System**:
- [frontend/src-tauri/src/audio/recording_manager.rs](frontend/src-tauri/src/audio/recording_manager.rs) - Recording orchestration
- [frontend/src-tauri/src/audio/pipeline.rs](frontend/src-tauri/src/audio/pipeline.rs) - Audio mixing and VAD
- [frontend/src-tauri/src/audio/recording_saver.rs](frontend/src-tauri/src/audio/recording_saver.rs) - Audio file writing

**UI Components**:
- [frontend/src/app/page.tsx](frontend/src/app/page.tsx) - Main recording interface
- [frontend/src/components/Sidebar/SidebarProvider.tsx](frontend/src/components/Sidebar/SidebarProvider.tsx) - Global state management

**Whisper Integration**:
- [frontend/src-tauri/src/whisper_engine/whisper_engine.rs](frontend/src-tauri/src/whisper_engine/whisper_engine.rs) - Whisper model management and transcription

## Agent skills

### Issue tracker

Issues and PRDs are tracked as local markdown files under `.scratch/`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default canonical labels are used as-is (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context repo — one `CONTEXT.md` at the root, ADRs in `docs/adr/`. See `docs/agents/domain.md`.

## Development Constraints

### 开发流程 (最高原则)

3 阶段 + 正交过程纪律。旧 10 层体系的缺陷及改进理由见 [层 5 回顾](docs/adr/003-voiceprint-speaker-diarization.md)。

```
══════════════════════════════════════════════════════
                    过程纪律 (正交于所有阶段)
══════════════════════════════════════════════════════
  TDD 纪律 · Agent 隔离 · 提交规范 · 逐循环前置检查
  Spec 就绪 Gate · 1.5h 时限 · 禁止改测试意图
  单 commit 单逻辑单元 · Agent 上下文防污染
══════════════════════════════════════════════════════

  阶段 A — 设计 (自上而下产出，自下而上校验)
  ┌────────────────────────────────────────────────┐
  │ A1. 需求定义    │ 用户可见的功能清单、优先级   │
  │                 │ 输出: FR 列表 + 补充需求      │
  ├────────────────────────────────────────────────┤
  │ A2. 架构决策    │ 服务拓扑、数据归属、管线结构 │
  │                 │ 输出: ADR + API 端点职责      │
  ├────────────────────────────────────────────────┤
  │ A3. 详细设计    │ 字段类型、边界行为、算法参数 │
  │                 │ 输出: API 契约 + ADR 细化     │
  │    ↓ 回溯校验   │ A3→A2→A1 反向检查矛盾        │
  └────────────────────────────────────────────────┘

  阶段 B — 实现 (TDD 循环，单模块逐一推进)
  ┌────────────────────────────────────────────────┐
  │ B1. RED    │ Agent A: 写测试 + 骨架 → commit   │
  │ B2. GREEN  │ Agent B: 最小实现 → commit        │
  │ B3. REFACTOR│ Agent A: 同文件内重构 → commit   │
  │    ↓ 模块   │ 单模块 3-commit 循环完成          │
  └────────────────────────────────────────────────┘

  阶段 C — 集成 & 交付
  ┌────────────────────────────────────────────────┐
  │ C1. 接线     │ 模块间集成 → commit             │
  │ C2. 冒烟     │ E2E 10s 录音冒烟 gate            │
  │ C3. 系统测试 │ 完整用户路径 E2E                  │
  │ C4. 打包     │ start.sh / start.bat             │
  └────────────────────────────────────────────────┘
```

**与旧 10 层的对应关系**：

| 旧层 | 新结构 | 说明 |
|---|---|---|
| 层 1 方法论 | 过程纪律 (正交) | 不是"一层"，而是贯穿所有阶段 |
| 层 2 开发条件 | 过程纪律 (正交) | 进入 B 阶段前的操作门禁 |
| 层 3 需求 | A1. 需求定义 | 边界清晰：需求=功能清单 |
| 层 4 架构 | A2. 架构决策 | 严格限"服务拓扑+数据归属+管线结构"，API 字段类型归 A3 |
| 层 5 细节 | A3. 详细设计 | 字段类型/边界行为/算法参数。旧层 4 中越界的细节迁入此处 |
| 层 6 + 层 7 | B 阶段 (TDD 循环) | RED→GREEN→REFACTOR 是一体循环，模块验证在其中完成 |
| 层 8 + 层 9 | C2 + C3 | 集成冒烟 + 系统测试 |
| 层 10 | C4 | 打包 |

### 回溯校验方法 (阶段 A 内部 — grill-with-docs 操作纪律)

**自下而上验证：先给建议，再回溯检查上层。**

传统自上而下 (A1 定框 → A3 服从) 的问题：假设上层一定正确，下层发现矛盾时倾向于服从上层，错失了暴露上层错误的窗口。

**正确方法**：

```
给出 A3 建议
    ↓
回溯检查 A2, A1
    ↓
发现矛盾 → 两边都可能错 → 独立论证哪个对
```

**三类结果**：

| 回溯结果 | 含义 | 动作 |
|---|---|---|
| 无矛盾 | 上下对齐 | 采纳建议 |
| 有矛盾，但 A3 对 | 上层决策有问题 | 修正上层，再采纳建议 |
| 有矛盾，但上层对 | A3 建议有问题 | 修正 A3 建议 |

**与 TDD 映射**：
- 阶段 A 讨论 (grill-with-docs) 使用本方法 — 建议先于约束检查
- 阶段 B RED Agent A 使用 Spec 就绪 Gate — spec 先于测试

本质是同一个原则 (先产出再校验)，分别在设计阶段和实现阶段落地。

### 逐循环前置检查 (过程纪律 — B 阶段门禁)

**此检查由人执行，不是 AI agent。** 每个 TDD 循环前 30 秒。

```
1. git status           — 工作区干净？在正确分支？
2. git log -3           — 上一循环的 commit 在 log 中可读？
3. 读 issue 文件         — 子任务描述够写测试？(粗判)
4. 编译/测试发现验证      — Rust: cargo build && cargo test --no-run
                            Python: pytest --co
5. 新 Agent 窗口         — 新 session、无上一循环残留上下文
6. TDD 编号已填入        — 提示词模板中 [TDD编号] 已替换
```

前 4 条是硬 gate，任一条不过 → 不开工。后 2 条是操作纪律。

**循环失败恢复**：上一循环升级后，重新执行全部 6 条。升级报告不放工作区，开新窗口时附带升级报告的路径即可。

### Spec 就绪 Gate (过程纪律)

Agent A 在写 RED 测试前，检查 issue 文件中是否包含：

- **输入数据格式** — 函数签名、参数类型、数据结构
- **预期输出格式** — 返回值、副作用、error 类型
- **至少一个 concrete example** — 给定 X → 期望 Y

缺任一项 → Agent A 输出 `[HUMAN ESCALATION] Spec incomplete: <具体缺什么>`，不自己猜测补充。

### 操作基础设施 (过程纪律)

**人的完整操作手册**: `.scratch/funasr-migration/HUMAN-GUIDE.md`。Agent 无需读此文件。

#### 交接文件

Agent A 完成 RED commit 后，将交接包写入文件，**不靠人复制粘贴**：
- 路径: `.scratch/funasr-migration/tdd/{TDD编号}-handoff.md`
- Agent B 的提示词模板引用该文件路径

#### 升级报告

Agent 升级后，升级报告写入文件：
- 路径: `.scratch/funasr-migration/tdd/{TDD编号}-escalation.md`
- RED commit **保留**在 git history 中，不回退
- 人工解决后，新的 RED commit 替代上一个

#### 分支策略

- 每个 issue 一条分支。Issue 内所有 TDD 循环 commit (RED+GREEN+REFACTOR) 都堆在同一条分支上
- 人类只在 issue 全部完成后 merge 到 main，不分步 merge
- `git bisect` 视角：一个 commit = 一个可编译的状态点

#### Issue 上下文文件 — 知识索引

每个 issue 维护一个上下文文件，**作为 Agent 的导航锚点**。Agent 打开后第一眼看它而非从头啃 5 个文件。

- 路径: `.scratch/funasr-migration/tdd/{issue-slug}-context.md`
- 由人类在 issue 启动时初始化

**文件格式**：

```
# context: {issue-slug}

## 我的位置
Issue {N} / 当前 TDD: {编号} / 上一步: {上一步状态}

## 我需要知道什么

| 我想知道 | 去这里 |
|---|---|
| 领域术语 | ../../../../CONTEXT.md |
| 架构决策 | ../../../../docs/adr/003-... |
| 本次要建什么 | ../issues/{N}-{slug}.md |
| API 契约 | ../../../../docs/api/{funasr\|fastapi}.md |
| 前几个 TDD 做了什么 | (本文件 — 历史记录) |

## 历史记录
[{TDD编号}] [状态] 测试意图: ... | 实现策略: ... | 重构: ...
```

**每次循环后**：Agent 在历史记录栏追加一行。下一个 Agent 打开时先读本文件，再按索引定向读具体文件，不从零啃 5 个文件。

### 功能需求 (A1 — 需求定义)

优先级从高到低排列。上方未完成时不启动下方。

| 优先级 | # | 需求 | 核心交付 |
|---|---|---|---|
| P1 | 1 | **实时中文转录** | FunASR SenseVoiceSmall，每 2s 固定 chunk，返回 segments |
| P2 | 2 | **声纹注册** | 固定文本朗读，CAM++ 256 维向量，FunASR 侧 SQLite |
| P3 | 7 | **摘要生成** | Ollama qwen3:8b，用户手动触发 |
| P4 | 9 | **一键启动脚本** | Windows .bat + Debian .sh，自动安装依赖 |
| P5 | 10 | **离线音频导入** | 中文音频文件 → 转录 + 说话人分离 + 摘要 |
| P6 | 6 | **崩溃容灾** | 断路器 + 渐进式健康探测 + resume_from 续传 |
| P7 | 3 | **实时说话人标注** | 3 级决策：≥0.7 匹配 / k-means 聚类 / 自适应扩容 |
| P8 | 4 | **会后命名** | 聚类标签 → 姓名，命名后落 DB，sidecar 防丢 |
| P9 | 5 | **声纹导入/导出** | v1 明文 JSON，v2 加密 |
| P10 | 8 | **Markdown 导出** | `**姓名** (HH:MM:SS): 内容` 格式 |
| P11 | 11 | **双路混音** | 麦克风 + 系统音频同时采集、RMS 混音。录音 48kHz 双路存档 + 16kHz 混音送转录 |

**补充需求** (不属于 P1-P11，但架构必须反映)：

- **录音格式**: 48kHz WAV，边录边写 (chunk-by-chunk 追加，非内存缓冲)。录音结束时才 flush 会导致崩溃后文件损坏。双路存档 + 16kHz 混音送转录。
- **录音生命周期**: 用户选择磁盘路径。录音文件**永不自动删除**——录音是最终真相源，安全优先。
- **旧数据兼容**: 不需要。FunASR 迁移是全新起点，旧 meetings 表结构直接变更。
- **离线导入格式**: 依赖 ffmpeg 做格式转换 (MP3/M4A/... → 16kHz mono WAV)。
- **声纹注册文本**: "北风与太阳" (标准中文语音学朗读文本，~150 字，覆盖全音素)。
- **隐私边界**: 零遥测、零云端、零自动更新检查。纯本地。
- **长会议 segments 安全**: 每 5 分钟增量追加 segments 到 sidecar。Tauri 崩溃也不丢已转录内容。sidecar 是 segments 恢复的唯一载体，无需数据库记录中转。
- **ffmpeg**: 随应用自带 (Tauri bundle)，不依赖系统安装。离线音频导入时自动调用格式转换。

### 运行环境

- **全本地部署** — FunASR (:8178) + FastAPI (:5167) + Ollama (:11434) 均本机运行
- **平台** — Windows + Debian
- **Python 3.9+** — FunASR + FastAPI
- **Ollama** — 摘要用，需 qwen3:8b 模型
- **GPU** — FunASR 受益但非必须 (CPU 可跑)
- **Tauri 2.x + Next.js 14 + React 18** — 前端栈不变
- **会议时长** — 常超 2h。Session 超时 ≥ 4h。k 默认 3
- **同时说话人数** — 一般 ≤ 7，极端 ≤ 10+。自适应扩容必须覆盖 3→10 的跨度
- **转录延迟** — < 3s (chunk 到达 → segment 返回)
- **模型磁盘** — 无限制 (SenseVoiceSmall ~230MB)

### 编程约束 (过程纪律)

1. **严格 TDD** — RED → GREEN → REFACTOR，一个 TDD 循环 = 一个 git commit。小步快跑，一个循环不涉及太多变更
2. **Agent 隔离** — 写测试的 agent 与写实现的 agent 分离，禁止同一 agent 自测自写。一个 agent 写测试并 commit (RED)，另一个 agent 读测试、写实现并 commit (GREEN)，交替进行，始终保持测试与实现由不同上下文窗口产出。REFACTOR 阶段由原测试作者 (Agent A) 执行——如发现测试被 Agent B 修改过，回退实现 commit
3. **单线程顺序执行** — 同一时刻只有一个 agent 在写代码，不与 Agent 隔离冲突——隔离 = 分工，单线程 = 并发度
4. **TDD 硬性时限** — 每个 TDD 循环 ≤ 1.5h。超时 → agent 输出失败报告 (卡在哪里、缺什么信息)，人工介入。不回退，不自行改测试降难度
5. **不跳过 TDD** — 需要人工帮助时明确提示，不找任何理由绕过 TDD 流程
6. **禁止改测试意图** — Agent B (实现者) 不得修改 assert 语义、测试逻辑、输入输出期望。但以下机械适配 Agent B **可以且必须修正**：import 路径错误、函数名拼写不匹配、类型签名偏差。修正后必须在 commit message 注明。测试过不了就找人，不自改
7. **单 commit 单逻辑单元** — 每个 commit `cargo build` 必须通过，git bisect 友好
8. **不并行保留新旧引擎** — git revert 即回归，不维护 Whisper/Parakeet 兼容路径
9. **Rust 不持有 SQLite** — 持久化通过 FunASR/FastAPI API 或 JSON sidecar
10. **网络** — Debian，可连外网，代理端口 socks(7898) / http(7899)
11. **Agent 上下文防污染** — 每个 agent 启动时必须读：ADR-003、CONTEXT.md、当前 issue 文件、上游依赖 issue 文件。用三句话简述自己的理解后再动代码。禁止跳过
12. **Issue-0 环境门禁** — pip install funasr + cargo build + cargo test 空跑 + pytest 空跑 全部通过后，才能开始 Issue A
13. **粒度验证** — 第一个 agent 窗口拿最复杂的 TDD (03b k-means) 跑完整循环，回流实际时长/token，调整其余估时
14. **集成冒烟 gate** — B4 (接线 commit) 后必须跑 E2E 冒烟：录音 10s → 前端出中文。不通过则 B1-B3 不算完成
15. **工具链验证** — Issue-0 确认 `cargo test` 增量编译 < 60s，否则配 sccache。Python 用 `pytest --lf` 只跑失败

### TDD 操作规范 (过程纪律 — B 阶段核心)

> **阅读顺序**: 先读 A (RED 标准) ← 入口规范，再读 G (失败报告) 和 I (人工升级) ← 安全阀，然后按需翻阅其余。B/C/D/E/F/H 为执行细节。

#### A. RED 标准 — 合格的"测试先红"

不合格的红 (Agent A 不得提交)：
- `assert 1 == 2` — 无意图，import 后就能绿
- `import nonexistent_module` — 语法红，加文件就绿
- `assert result is not None` — 太弱，空实现返回值就绿

合格的红必须同时满足：
- 断言击中本 TDD 子任务的目标行为（不是通用断言）
- 失败原因 = 功能还没实现（不是 import 错误、语法错误、类型错误）
- 一旦功能正确实现，该断言自然通过

**Agent A 提交前必须实际运行测试并看到红** — 不得跳过运行直接 commit。运行失败原因必须是"目标功能尚未实现"，不是 import 错误或语法错误。

**测试文件位置约定** (Agent A 在 RED commit 中确定)：
- Rust: `src/audio/chunk_accumulator.rs` (实现) ↔ `src/audio/chunk_accumulator.rs` 内嵌 `#[cfg(test)] mod tests`
  - 新建模块：Agent A 创建 `src/<mod>/<file>.rs` 的骨架（仅 `pub struct`/`pub fn` 签名），测试写在同文件 `#[cfg(test)]` 内
- Python: `services/funasr/speaker_clusterer.py` (实现) ↔ `services/funasr/tests/test_speaker_clusterer.py`
  - Agent A 创建空的实现文件骨架（仅 class/def 签名 + `pass`），测试文件 import 该骨架

Agent A 的 RED commit 包含: 测试文件 + 实现文件骨架（仅签名，无逻辑）。Agent B 拿到后只需在骨架中填逻辑。

#### B. GREEN 标准 — 合理的最小实现

- 禁止硬编码测试答案。例：测试 `sum([1,2,3]) == 6`，可以写 `sum(iterable)` 但不能 `return 6`
- 可以用通用逻辑处理当前测试的所有输入，不需要处理未覆盖的边缘情况
- 边缘情况留给下一轮 TDD

#### C. REFACTOR 范围 — 保守级

Agent A 只能做"让前面两个人工作像一个人写的"：
- 同文件内：重命名变量、提取私有函数、消除重复、用 idiomatic 写法
- 禁止：改 API 签名、改返回类型、跨文件移动代码、重构模块结构
- 更大重构 → 记录为下一轮 TDD 的输入，不自作改动

#### D. Commit 规范 — TDD 可追溯性

每循环 3 个 commit，格式：`[阶段] TDD编号 模块: 目标行为`

```
[RED] 03b k-means: failing test for adaptive cluster expansion
[GREEN] 03b k-means: implement add_embedding with k+1 expansion
[REFACTOR] 03b k-means: extract centroid distance helper
```

#### E. 不需要 TDD 的内容

以下跳过 TDD，直接实现：

| 类型 | 示例 | 原因 |
|---|---|---|
| 纯删除 | B5-B10 (删 whisper_engine 等) | 无新行为可测 |
| 集成接线 | B4 (改管线接 FunASR)、C1-C5 | 结构变更，正确性由 E2E 冒烟验证 |
| 配置脚本 | Issue-0、start.sh/start.bat | 纯环境操作 |
| 前端 UI 布局 | 声纹注册页、命名弹窗 | 视觉组件，pytest/cargo test 无法测 |

#### F. 测试质量标准

每个 TDD 子任务的测试至少覆盖：

| 必须 | 示例 (03b k-means) |
|---|---|
| Happy path | `add_embedding(emb)` → 分配到最近质心 |
| 至少一个边界 | 空聚类时第一个 embedding → 创建第一个质心 |
| 至少一个错误路径 | 超过 k 个说话人时触发自适应扩容 k+1 |

不满足 → Agent A RED commit 不合格，回退重写。

#### G. Agent B 失败报告格式

在 1.5h 内无法 GREEN 时，输出：

```
### 失败报告 — [TDD编号]

**失败的 assert**: <具体断言行>
**当前实现策略**: <一句话>
**怀疑点**:
  - 测试可能有问题: <原因 / 无>
  - 实现不够: <原因 / 无>
  - 缺信息: <需要什么>
**当前代码能编译/通过 import**: <是/否>
**建议**: <人工往哪个方向看>
```

禁止模糊描述（"不太行"、"有 bug"）——必须指向具体 assert 或具体缺失信息。

#### H. Agent A → Agent B 交接包

Agent B 拿到的不只是红测试文件：

```
### Agent A 交接

**TDD 编号**: 03b
**测试意图**: <一句话>
**关键假设**: <Agent A 写了测试基于什么假设>
**输入/输出约定**: <具体的数据进出>
**已知不足**: <什么没测，留给下一循环>
```

Agent B 回复"理解：[一句话重述意图]"后才能开始写实现。

#### I. 人工升级触发器

以下情况立即停止、报告 `[HUMAN ESCALATION]`、不自己猜：

| 触发条件 | 典型信号 |
|---|---|
| Spec 模糊 | 测试写什么不清楚、需求在 issue 里找不到对应描述 |
| 环境异常 | pip install 失败、cargo build 报非代码错误、端口被占 |
| 两难取舍 | 两个合理方案互斥、需要拍板但不在 issue spec 里 |
| 连续两次 GREEN 失败 | Agent B 第一轮写错，第二轮又错 — 第三次大概率退化 |
| 改动超出范围 | 需要改 issue 未列出的文件、删不该删的 pub API |
| 测试与代码矛盾 | Agent B 认为测试的输入/输出在真实环境中不可能 |
| Agent A 卡住 | 无法在 1h 内写出合格的 RED 测试 (spec 模糊 / 不知道测什么 / 无法创建骨架) |
