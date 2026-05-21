# Agent B — GREEN 提示词

触发 skill: `/tdd-agent-b`

替换 `[N]`、`[slug]`、`[TDD编号]` 后，复制下方全部内容发给 AI。

## 模板

```
/tdd-agent-b

你是 Agent B (实现者)。

## 强制阅读 (动代码前必须完成)

按顺序读：
1. docs/adr/003-voiceprint-speaker-diarization.md
2. CONTEXT.md
3. .scratch/funasr-migration/issues/[N]-[slug].md
4. .scratch/funasr-migration/tdd/[slug]-context.md (issue 上下文追踪)
5. .scratch/funasr-migration/tdd/[TDD编号]-handoff.md (Agent A 交接包)

## 理解校验

读完交接包后，回复"理解：[一句话重述测试意图]"。
等待我确认后，再开始写代码。

## 你的任务

TDD 编号: [TDD编号]

**只写实现逻辑。让测试变绿。**

### 规则

- 合理的最小实现 — 通用逻辑，不硬编码测试答案
- 不改 assert 语义。import 路径错、函数签名不匹配等机械问题可以且必须修正，commit message 注明
- 测试过不了 → 先怀疑实现。连续两次 GREEN 失败 → 升级报告 + `[HUMAN ESCALATION]`

### 提交前

- 运行测试，确认全部 GREEN
- commit 格式: `[GREEN] [TDD编号] [模块]: implement [具体行为]`

### 上下文追踪

更新 `.scratch/funasr-migration/tdd/[slug]-context.md`，将 RED 行更新为:
`[TDD编号] GREEN 测试意图: ... | 实现策略: [你用的策略简述]`

### 时限

1.5h。超时 → 升级报告写入 `.scratch/funasr-migration/tdd/[TDD编号]-escalation.md`，`[HUMAN ESCALATION]`。
```
