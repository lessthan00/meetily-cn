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

3. **Markdown 导出格式**：必须输出如下对齐格式 (Q12)：
   ```markdown
   # 会议标题 — 2026-05-23

   ## AI 摘要
   ...摘要文本... (已生成时展示，未生成则跳过此 section)

   ## 完整转录
   **张甜甜** (00:00:00): 我认为这个方案可行。
   **说话人 A** (00:00:05): 但是有一些问题...
   **未知** (00:00:10): ... (speaker=null 时显示"未知")
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

## grill-with-docs 确认 (2026-05-20, 更新 2026-05-23)

- **两阶段 commit**: 第一阶段新建端点，第二阶段删除旧端点
- **`GET /get-meeting/{id}` 改造**: 读 `meetings.transcript_json` 列返回 segments；无则返回空数组
- **旧表删除**: 六张旧表在第二阶段删除
- **save-transcript 容灾 (Q10 补充)**: Rust 调 save 失败 → sidecar 保留 → 启动时自动重试
- **自动标题生成 (Q17)**: `/session/end` 后自动调 `POST /meetings/{id}/generate-title` → Ollama 生成 ≤ 10 字标题 → 写入 meetings.title。失败静默降级
- **摘要 prompt (Q16)**: FastAPI 侧维护中文 system prompt (概述/决议/待办)。用户手动触发
- **Ollama 安装引导 (Q16 补充)**: 未安装 → 前端横幅 + 安装链接。模型未下载 → 「下载模型」按钮 + 进度条。状态检测: `GET /meetings/{id}/summary-status`
- **Markdown 含摘要 (Q12)**: 已生成摘要时展示 `## AI 摘要` section，未生成则跳过。speaker=null → `**未知**`

## Blocked by

None — 可独立开工（仅依赖现有 backend/app/main.py，无需 #01-#05 的变更）