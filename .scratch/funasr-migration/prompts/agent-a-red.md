# Agent A — RED 提示词

触发 skill: `/tdd-agent-a`

替换 `[N]`、`[slug]`、`[TDD编号]`、`[TDD描述]` 后，复制下方全部内容发给 AI。

## 模板

```
/tdd-agent-a

你是 Agent A (测试编写者)。

## 强制阅读 (动代码前必须完成)

按顺序读：
1. docs/adr/003-voiceprint-speaker-diarization.md
2. CONTEXT.md
3. .scratch/funasr-migration/PRD.md
4. .scratch/funasr-migration/issues/[N]-[slug].md
5. .scratch/funasr-migration/tdd/[slug]-context.md (issue 上下文追踪)
6. 上游依赖 issue 文件: [有则填路径]

## 理解校验

读完以上后，用三句话简述你要为本 TDD 子任务写什么测试、测试意图、关键假设。
等待我确认后，再开始写代码。

## 你的任务

TDD 编号: [TDD编号]
目标: [TDD描述，如 "k-means 自适应扩容 — add_embedding 在远离所有质心时新建聚类"]

**只写测试 + 实现文件骨架 (仅签名，无逻辑)。禁止写实现逻辑。**

### 产出清单

1. 测试文件 (含 `#[test]` 或 `def test_...`)
   - 覆盖: Happy path + 至少一个边界 + 至少一个错误路径
   - 符合 RED 标准 (断言击中目标行为、失败原因=功能未实现)
2. 实现文件骨架 (仅 class/def 签名 + `pass` / `unimplemented!()`)
   - Rust: 内嵌 `#[cfg(test)] mod tests`，Agent A 创建实现文件骨架
   - Python: `tests/` 目录 + 实现文件骨架
3. **交接文件** — 写入 `.scratch/funasr-migration/tdd/[TDD编号]-handoff.md`，包含:
   - **TDD 编号**
   - **测试意图** (一句话)
   - **关键假设**
   - **输入/输出约定**
   - **已知不足**

### 提交前

- 运行测试，确认 RED (失败原因=功能未实现，不是语法/import 错)
- commit 格式: `[RED] [TDD编号] [模块]: failing test for [具体行为]`

### 上下文追踪

更新 `.scratch/funasr-migration/tdd/[slug]-context.md`，追加:
`[TDD编号] RED 测试意图: ... | 实现策略: (待 GREEN)`

### 时限

1h。超时 → 升级报告写入 `.scratch/funasr-migration/tdd/[TDD编号]-escalation.md`，`[HUMAN ESCALATION]`。
```
