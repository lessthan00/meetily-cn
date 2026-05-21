# Issue C — TDD Progress (2026-05-20)

## State: GREEN Phase in progress (not yet confirmed by user)

## Completed

### Python — FunASR Service (RED → GREEN → REFACTOR) ✅
- **Test file**: `services/funasr/tests/test_integration_endpoints.py` — 26 tests covering all new endpoints
- **Implementation**: `services/funasr/app.py` — full rewrite with all ADR-003 endpoints wired to Issue A pure logic
- **95/95 tests pass** (69 existing + 26 new)

Endpoints implemented:
- POST /enroll (with audio validation, CAM++ embedding, DB storage)
- GET /speakers (list without embedding)
- DELETE /speakers/{id}
- POST /speakers/export (AES-256-GCM encrypted)
- POST /speakers/import (decrypt + merge)
- POST /session/start (k-means init)
- POST /session/end (clustering summary, session destroy)
- POST /process-audio?mode=realtime (extended with session_id + speaker labeling)
- POST /identify (standalone)
- POST /transcribe (standalone)
- GET /config, POST /config
- GET /health (existing, preserved)

Refactor done: removed `_pack_for_db` duplication (imports from db.py), simplified `export_speakers`.

### Rust — FunasrClient RED Tests ✅
- Added `wiremock = "0.6"` to `Cargo.toml` dev-dependencies
- Added 8 HTTP mock tests in `funasr_client.rs` (will fail to compile until methods exist)
- Tests cover: enroll, list_speakers, delete_speaker, start_session, end_session, process_audio with session_id, export_speakers, import_speakers

### Rust — FunasrClient GREEN Implementation ✅ (not compiled)
- Added new types: EnrollResponse, SpeakerInfo, ClusterSpeakerSummary, SessionEndResponse, ImportResponse, IdentifyResponse
- Added new methods: enroll(), list_speakers(), delete_speaker(), export_speakers(), import_speakers(), start_session(), end_session(), process_audio(with session_id), get_config(), update_config(), identify(), health_check()
- Modified transcribe() to delegate to process_audio(wav_bytes, None)

## NOT YET DONE

### Rust — apply_speaker_names command
- Need to add `apply_speaker_names` Tauri command in `recording_commands.rs`
- Need to register it in `lib.rs`
- Need to modify `stop_recording` to call `/session/end` and emit `session-ended`
- Need to modify `start_recording` to call `/session/start` and pass session_id to transcription task

### Rust — Session integration in recording flow
- `start_funasr_transcription_task` needs to accept and pass session_id
- `stop_recording` needs to call `end_session()` after transcription task stops

### Can't compile Rust on this machine
- Missing system library: `gdk-3.0` (GTK dev headers)
- `cargo check` / `cargo test` won't work until `libgtk-3-dev` is installed

### Frontend (React) — NOT STARTED
- Voiceprint enrollment page
- Speaker management page
- k-value input before meeting
- Transcript panel with speaker labels
- Post-meeting naming UI
- save-transcript call moved to after naming

## Next Steps (after user confirms)

1. Finish Rust GREEN: compile on user's machine, fix any issues
2. Add `apply_speaker_names` command
3. Wire session start/end into recording flow
4. TDD Step 4 (REFACTOR) for Rust
5. TDD Step 5: Confirm all tests pass
6. Frontend implementation
7. Register all new Tauri commands in lib.rs
