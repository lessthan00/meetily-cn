# context: {issue-slug}

复制此模板，替换 `{...}` 后保存为 `{issue-slug}-context.md`。

```
# context: {issue-slug}

## 我的位置
Issue {N} / 当前 TDD: {编号} / 上一步: (尚未开始)

## 我需要知道什么

| 我想知道 | 去这里 |
|---|---|
| 领域术语 (segment 是什么) | ../../../../CONTEXT.md |
| 为什么选 FunASR 替换 Whisper | ../../../../docs/adr/003-voiceprint-speaker-diarization.md |
| 本次要建什么 | ../issues/{N}-{slug}.md |
| API 契约 (请求/响应格式) | ../../../../docs/api/funasr.md |
| (FastAPI 侧 API) | ../../../../docs/api/fastapi.md |
| 完整实施计划 | ../PRD.md |
| 前几个 TDD 做了什么 | (本文件 — 历史记录) |

## 历史记录
(每个 TDD 循环后 Agent 自动追加一行)
```

## 使用说明

1. 替换 `{issue-slug}` 为 issue 文件名 slug（如 `02-funasr-transcription-pipeline`）
2. 替换 `{N}` 为 issue 编号（如 `02`）
3. 替换 `{编号}` 为第一个 TDD 编号（如 `02c`）
4. 如果某个 API 契约文件尚未创建，删除对应行
5. 如果 PRD 不需要（小 issue），删除对应行
