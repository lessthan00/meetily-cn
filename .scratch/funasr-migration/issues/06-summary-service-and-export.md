# 06 — 摘要服务验证 + Markdown 导出对齐

**Status**: ready-for-agent

## Parent

ADR-003: 声纹识别与说话人分离架构

## What to build

验证现有的摘要后端服务（:5167）是否与 ADR-003 的新架构对齐，修复不匹配处，确保 Markdown 导出包含说话人标注的转录文本。

端到端行为：会议录音完成 → 用户完成说话人命名 → Rust POST /save-transcript 保存 segments → 回到会议详情页 → 自动调用 `POST /generate-title`（轻量 prompt，10 字以内标题，~1s）→ 标题从默认日期时间替换为 AI 生成标题 → 用户手动点击「生成摘要」→ 前端调 `POST /process-transcript {meeting_id}` → FastAPI 从 `transcript_json` 列读取 segments → Ollama qwen3:8b 生成摘要 → 写入 `summary_json` 列 → 摘要展现在详情页 → 用户点击导出 → Markdown 文件包含 `**说话人姓名** (时间戳): 内容` 格式的转录 + AI 摘要。

> **关键决策**:
> - 会议开始时默认标题为日期时间（如「2026-05-19 14:30」）
> - 会后自动调用 Ollama 生成标题（轻量，不阻塞）
> - 摘要不自动生成——用户手动触发。避免每次录音结束都占用 GPU 资源，且并非每次都需要生成摘要

### 需要验证/修复的点

1. **摘要端点输入格式**：`POST /process-transcript` 接收 `{meeting_id}`，从 `meetings.transcript_json` 列读取 segments (已含 speaker 字段)，无需请求体传 segments。内部数据结构：
   ```json
   // transcript_json 列内容
   [
     {"start": 0.0, "end": 2.5, "text": "...", "speaker": "张甜甜"},
     {"start": 2.5, "end": 5.0, "text": "...", "speaker": "说话人 A"}
   ]
   ```

2. **Ollama 摘要 prompt**：是否需要更新提示词以利用说话人信息？当前 prompt 可能只针对纯文本设计

3. **Markdown 导出格式**：必须输出如下对齐格式：
   ```markdown
   # 会议记录 — 2026-05-17 10:00
   
   ## AI 摘要
   ...摘要文本...
   
   ## 完整转录
   **张甜甜** (00:00:00): 我认为这个方案可行。
   **说话人 A** (00:00:05): 但是有一些问题...
   ```

4. **会议详情页**：`frontend/src/app/meeting-details/` 确保展示摘要 + 标注说话人的转录

### 不在此 scope 内

- 不修改 Ollama 模型选择（保持 qwen3:8b）
- 不新增摘要功能（仅对齐现有功能）
- 不做摘要质量的 Prompt 工程优化

## Acceptance criteria

- [ ] `POST /process-transcript` 接收 `{meeting_id}`，从 `meetings.transcript_json` 列读取 segments 生成摘要
- [ ] 摘要生成成功并返回前端（`backend/examples/run_summary_workflow.py` 可作为测试脚本）
- [ ] Markdown 导出包含说话人标注（`**姓名** (HH:MM:SS): 内容`）
- [ ] 导出文件在 Typora/VS Code 中正确渲染
- [ ] 会议详情页同时显示摘要和完整转录
- [ ] 摘要生成失败时优雅降级（显示"摘要生成失败，请重试"）

## grill-with-docs 确认 (2026-05-20)

- **两阶段 commit**: 第一阶段新建端点（`POST /meetings`, `POST /save-transcript`(新), `POST /process-transcript`(新), `POST /generate-title`, `GET /export/{id}?format=md`），第二阶段删除旧端点（`GET /get-summary`, `GET /get-model-config`, `POST /save-model-config`, `GET /get-transcript-config`, `POST /save-transcript-config`, `POST /get-api-key`, `POST /get-transcript-api-key`, `POST /save-meeting-summary`, `POST /search-transcripts`）
- **`GET /get-meeting/{id}` 改造**: 读 `meetings.transcript_json` 列返回 segments；无则返回空数组（不 fallback 到旧表——旧表已在第二阶段删除）
- **旧表删除**: `transcripts`/`transcript_chunks`/`summary_processes`/`settings`/`transcript_settings`/`speaker_embeddings` 六张表在第二阶段连同旧端点一并 `DROP TABLE`。`speaker_embeddings` 从 FastAPI 侧删除是因为其正确归属在 FunASR 侧的 `services/funasr/data.db`
- **save-transcript 容灾**: Rust 调 `POST /save-transcript` 失败 → segments 写入 sidecar (`named_segments` 字段)。Rust 启动/健康恢复后扫描 sidecar → 自动重试保存

## Blocked by

None — 可独立开工（仅依赖现有 backend/app/main.py，无需 #01-#05 的变更）