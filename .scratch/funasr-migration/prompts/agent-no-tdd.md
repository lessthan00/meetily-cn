# 非 TDD 任务提示词

触发 skill: `/meetily-no-tdd`

用于纯删除、集成接线、配置脚本、前端 UI 布局。

## 模板

```
/meetily-no-tdd

本任务不适用 TDD。

## 强制阅读

按序读：
1. docs/adr/003-voiceprint-speaker-diarization.md
2. CONTEXT.md
3. .scratch/funasr-migration/PRD.md
4. .scratch/funasr-migration/issues/[N]-[slug].md

## 理解校验

用三句话简述：做什么、为什么不需要 TDD、如何验证正确性。
等待我确认后开始。

## 规则

- 单 commit 单逻辑单元
- 每个 commit 编译必须通过 (cargo build / python -m compileall)
- 验证方式: [E2E 冒烟 / 人工检查 / 脚本自检]
- 时限 1.5h，超时输出失败报告
```

## 示例 — 纯删除任务 (如 B5)

```
/meetily-no-tdd

本任务不适用 TDD (类型: 纯删除)。

阅读清单后简述: 删 whisper_engine 目录，修复 lib.rs 引用，纯删除无新行为，验证方式 cargo build 通过。

提交: [DELETE] B5 whisper_engine: remove whisper_engine crate module
```
