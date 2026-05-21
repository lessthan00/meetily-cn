# 提交前审查提示词

## 使用方式

在任务的所有测试通过、refactor 完成后，粘贴以下内容给 AI。

## 模板

```
审查工作区未提交的变更，产出 commit message，之后不再行动。

1. 用 generate_explanation 审查工作区 diff（相对于 HEAD 的所有未提交变更）：
   - `title`: 根据变更自动推断简述
   - `from_ref`: `HEAD`
   - `to_ref`: 省略（默认工作区）
   如果有严重发现，告知我并等待决定。

2. 根据审查结果和 diff 内容，给我一条 conventional commits 格式的 commit message：
   ```
   <type>(<scope>): <简短描述>

   <详细说明>
   ```
   类型选择：feat / fix / refactor / test

3. 以上完成后停止。不提交，不推送。
```

## 约束

- 只审查工作区相对于 HEAD 的变更，不扩散到仓库其余部分
- 审查发现严重问题时先报告，不自动修复
- 产出 commit message 后停止，由用户审核后再决定是否提交