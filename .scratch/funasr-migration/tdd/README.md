# TDD 子任务索引

将管线 issue 中的可独立测试逻辑拆分为 TDD 子任务，每个子任务在一小时内完成一个红-绿-重构循环。

## 任务总览

| 编号 | 名称 | 语言 | 父 Issue | 预计耗时 |
|---|---|---|---|---|
| [01a](01a-speaker-embedding-crud.md) | speaker_embeddings CRUD | Python | #01 | < 1h |
| [02a](02a-segment-deserialization.md) | segments JSON 反序列化 | Rust | #02 | < 45min |
| [02b](02b-wav-header-construction.md) | WAV header 构造 | Rust | #02 | < 1h |
| [02c](02c-wav-chunk-accumulator.md) | 2s chunk 累加器 | Rust | #02 (commit 02-1) | < 45min |
| [02d](02d-python-wav-decode.md) | Python WAV 解码 | Python | #02 (realtime/offline 组件拆分) | < 45min |
| [03a](03a-cosine-similarity.md) | 余弦相似度计算 | Python | #03 | < 45min |
| [03b](03b-kmeans-clustering.md) | k-means 质心分配 + 扩容 | Python | #03 | < 1.5h |
| [03c](03c-audio-validation.md) | 音频时长/质量校验 | Python | #03 | < 1h |
| [03d](03d-speaker-import-export.md) | 声纹 JSON 导入/导出 | Python | #03 | < 1h |
| [05a](05a-circuit-breaker.md) | 断路器状态机 | Rust | #05 | < 1.5h |
| [06a](06a-markdown-export-format.md) | Markdown 导出格式 | Python | #06 | < 45min |
| [06b](06b-summary-prompt-construction.md) | 摘要 prompt 构造 | Python | #06 | < 45min |

## 没有 TDD 子任务的 Issue

| Issue | 原因 |
|---|---|
| #02 commit 02-2 | 纯管线重构（删 VAD + 接 ChunkAccumulator），无新增可测试逻辑 |
| #02 commit 02-3 | 接线 FunASR（I/O 集成，非纯函数） |
| #04 删除旧转录代码 | 纯删除操作，拆为 6 次 commit 依次删除 |
| #07 一键启动脚本 + E2E | 纯 HITL 手动验证 |
| vad.py (fsmn-vad 封装) | 需要 FunASR 模型运行时，属于集成测试而非纯函数 TDD。在 Issue C 接线时随写随验 |

## 组件拆分架构 (grill-with-docs 确认)

FunASR 服务内部拆为 TDD 友好结构——公共组件为可独立测试的纯函数，两个 handler (realtime/offline) 各自组装：

```
services/funasr/
├── pipeline/
│   ├── wav.py          # WAV → PCM 解码 (TDD 02d)
│   ├── transcribe.py   # SenseVoiceSmall 封装 (集成，非 TDD)
│   ├── voiceprint.py   # CAM++ 封装 (集成，非 TDD)
│   ├── cluster.py      # SpeakerClusterer (TDD 03b 产物)
│   └── validate.py     # 音频校验 (TDD 03c 产物)
├── handlers/
│   ├── realtime.py     # 实时 handler: 解 WAV → fsmn-vad → 逐段转录+声纹 → 在线聚类 → 返回
│   └── offline.py      # 离线 handler: 解 WAV → fsmn-vad → 逐段转录+声纹 → batch 聚类 → 返回
├── db.py               # 声纹库 CRUD (TDD 01a 产物)
└── app.py              # FastAPI 路由: 读 ?mode= → dispatch 到对应 handler
```

## 执行顺序建议

```
第一轮（无依赖，全并行）:
  01a (CRUD) + 02a (反序列化) + 02b (WAV构造) + 02c (累加器) + 02d (WAV解码) + 03a (余弦)

第二轮（依赖 03a + 第三轮前置）:
  03b (k-means) + 03c (音频校验) + 03d (导入导出) + 06a (导出格式) + 06b (prompt) → 并行

第三轮（依赖 02a 的类型定义）:
  05a (断路器)
```

## 使用方式

每个 TDD 子任务文件包含：
- **RED** — 具体的失败测试用例（可直接复制运行）
- **GREEN** — 最小实现的大致结构
- **REFACTOR** — 优化的具体方向

Agent 或开发者应严格按 RED → GREEN → REFACTOR 顺序推进，不跳步。
