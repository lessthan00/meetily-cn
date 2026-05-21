# 提示词使用指南

## Skill → 提示词映射

| 触发命令 | 角色 | 职责 | 时限 | 提示词文件 |
|---|---|---|---|---|
| `/tdd-agent-a` | Agent A | 写测试 + 骨架 (RED) | 1h | [agent-a-red.md](agent-a-red.md) |
| `/tdd-agent-b` | Agent B | 写实现 (GREEN) | 1.5h | [agent-b-green.md](agent-b-green.md) |
| `/tdd-agent-a` | Agent A | 重构 (REFACTOR) | — | [agent-a-refactor.md](agent-a-refactor.md) |
| `/meetily-no-tdd` | — | 删除/集成/脚本/UI | 1.5h | [agent-no-tdd.md](agent-no-tdd.md) |

## 文件基础设施

| 文件 | 位置 | 用途 |
|---|---|---|
| 交接包 | `.scratch/funasr-migration/tdd/{TDD编号}-handoff.md` | Agent A 写入，Agent B 读取。不靠人复制粘贴 |
| 升级报告 | `.scratch/funasr-migration/tdd/{TDD编号}-escalation.md` | Agent 升级时写入 |
| 上下文追踪 | `.scratch/funasr-migration/tdd/{issue-slug}-context.md` | 每个 TDD 循环追加一行，防止 Agent 遗忘 |

## 分支策略

- 每个 issue 一条分支。Issue 内所有 TDD 循环 commit 堆在同一条分支上
- 人类只在 issue 全部完成后 merge 到 main
- RED commit 不回退 (升级后保留)，新 RED commit 替代上一个

## 完整 TDD 循环操作流程

### Issue 启动 (人做，一次性)

1. Issue-0 环境门禁已通过
2. 创建 context 文件: `.scratch/funasr-migration/tdd/{issue-slug}-context.md`
   - 填入"我的位置"和"我需要知道什么"表 (见 CLAUDE.md Issue 上下文文件格式)
   - 历史记录栏暂时留空

### 准备 (每个 TDD 循环前，人做)

1. 更新 context 文件的"我的位置"行 (当前 TDD 编号 + 上一步状态)
2. 选定下一个 TDD 子任务
3. 打开**两个独立的 AI agent 窗口**
4. 填写提示词模板
5. 执行前置检查清单 (见 CLAUDE.md)

### 循环执行 (3 步)

**Step 1 — RED (Agent A 窗口)**

1. 复制 `agent-a-red.md` 模板，粘贴到 Agent A 窗口
2. Agent A 理解校验后你确认
3. Agent A 产出: RED commit + `.scratch/funasr-migration/tdd/[TDD编号]-handoff.md` + 更新 context 文件
4. **你校验**: 运行测试，确认 RED；检查 handoff 文件存在且内容完整

**Step 2 — GREEN (Agent B 窗口)**

1. 复制 `agent-b-green.md` 模板，粘贴到 Agent B 窗口
2. Agent B 自动读取 handoff 文件和 context 文件 (模板已引用路径)
3. Agent B 理解校验后你确认
4. Agent B 产出: GREEN commit + 更新 context 文件
5. **你校验**: 运行测试，确认全绿

**Step 3 — REFACTOR (回到 Agent A 窗口)**

1. 复制 `agent-a-refactor.md` 模板，粘贴到 Agent A 窗口
2. Agent A 自动读取 context 文件
3. Agent A 理解校验后你确认
4. Agent A 产出: REFACTOR commit + 更新 context 文件
5. **你校验**: 运行测试，确认仍全绿；检查 Agent B 没改测试意图

循环结束 → 下一个 TDD 子任务

### 升级处理

Agent 输出 `[HUMAN ESCALATION]`：
- 升级报告在 `.scratch/funasr-migration/tdd/[TDD编号]-escalation.md`
- RED commit 保留在 git history，不回退
- 人工解决后，下一个 RED commit 替代上一个
- 重新执行前置检查清单

### 非 TDD 任务

触发 `/meetily-no-tdd`，用 `agent-no-tdd.md`。单窗口即可。

## 提示词填写清单

每次使用前替换:
- `[N]` — issue 编号
- `[slug]` — issue slug
- `[TDD编号]` — TDD 子任务编号
- `[TDD描述]` — 来自 PRD TDD 子任务表
- 上游依赖 issue 路径 — 来自 issue 文件 `Blocked by`

## 测试输出速查 (给人看的)

运行测试后只看最后几行：

**Rust (`cargo test`)**: 看最后一行

```
test result: ok. 3 passed; 0 failed    → ✅ GREEN，全部通过
test result: FAILED. 2 passed; 1 failed → ❌ RED，有 1 个测试失败 (正确 — 功能未实现)
error[E0432]: unresolved import        → ❌ 编译错 — Agent 的代码写坏了，不是 RED
error[E0601]: main function not found  → ❌ 编译错 — 同上
```

**Python (`pytest`)**: 看倒数 3 行

```
3 passed in 0.12s                        → ✅ GREEN
1 failed, 2 passed in 0.15s              → ❌ RED (功能未实现) — 正确
ImportError: No module named 'xxx'       → ❌ 环境错 — Agent 的 import 写错了
1 error in 0.10s                         → ❌ 编译错 — 代码有语法问题
```

**关键区分**：
- `failed` / `FAILED` = 测试跑了但断言失败 → 正常的 RED，通过
- `error` / `ImportError` / `E0432` = 测试没跑到 → 代码写坏了，不通过
- Agent 声称"RED"但输出是 error → NOT OK，Agent 在撒谎

## Context 文件初始化

复制模板 `.scratch/funasr-migration/tdd/CONTEXT-TEMPLATE.md`，见文件内说明。
