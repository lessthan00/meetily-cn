# Issue A — Python 纯逻辑 TDD

## 开始前必读（按顺序）

1. `.scratch/funasr-migration/issues/01-database-migration.md`
2. `.scratch/funasr-migration/issues/03-voiceprint-enrollment-and-identification.md`
3. `.scratch/funasr-migration/issues/06-summary-service-and-export.md`
4. `docs/adr/003-voiceprint-speaker-diarization.md`
5. `CLAUDE.md`
6. `CONTEXT.md`
7. `.scratch/funasr-migration/tdd/README.md`

## 任务

实现上述 issue 文件中所有标注为 Python 的 TDD 子任务。TDD 文件位于 `.scratch/funasr-migration/tdd/`，每个文件包含 RED → GREEN → REFACTOR 三步的具体测试用例和实现方向。

工作目录：`services/funasr/`

## 语言与框架

- Python 3.9+、pytest、aiosqlite
- 加密：cryptography (AES-256-GCM)

## 约束

- 只创建 `services/funasr/` 下的纯 Python 模块（函数/类），不创建 FastAPI 端点或 `main.py`
- 不启动 HTTP 服务
- 不碰 `backend/app/`、`frontend/`、任何 `.rs` 文件

## 完成标准

- `services/funasr/` 下 `pytest` 全部通过
- 每个 TDD 子任务走完 RED → GREEN → REFACTOR，每个循环一个 git commit
- 涉及的 issue 文件 acceptance criteria 全部打勾
