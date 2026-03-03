# 完整工作流概览

> 所有场景的 skill 调用顺序汇总。详细步骤见各场景文件。

---

## 两类 Skill

| 类型 | Skills | 作用 |
|------|--------|------|
| **流程类（Superpowers）** | brainstorming, writing-plans, tdd, debugging... | 控制「怎么做事」的纪律，不能跳过 |
| **领域类（Domain）** | git, db, security, test, perf, review | 具体技术参考手册，按需调用 |

---

## 场景速览

### 场景 1：开发新功能 → [详细](./01-new-feature.md)

```
brainstorming → writing-plans → tdd
  ↑ 过程中按需: security / db / test
→ verification → requesting-review → git → finishing-branch
```

### 场景 2：修复 Bug → [详细](./02-bug-fix.md)

```
systematic-debugging → tdd（写复现测试）→ 修复
  ↑ 过程中按需: db / perf
→ verification → git
```

### 场景 3：代码审查 → [详细](./03-code-review.md)

```
审查别人: review → security
收到反馈: receiving-code-review
```

### 场景 4：大型任务并行 → [详细](./04-parallel-tasks.md)

```
brainstorming → writing-plans → dispatching-parallel-agents / subagent-driven-development
  + using-git-worktrees（各任务独立 worktree）
→ verification + requesting-review + finishing-branch
```

---

## 全局触发速查

| 什么时候 | 用哪个 skill |
|---------|-------------|
| 接到新功能需求，准备动手前 | `brainstorming` → `writing-plans` |
| 写任何代码之前 | `tdd` |
| 遇到 bug / 测试失败 | `systematic-debugging` |
| 写认证、权限、输入验证 | `security` |
| 写 SQL / 设计表 / 慢查询 | `db` |
| 写测试 / 测试挂了 | `test` |
| 性能问题 | `perf` |
| 提交 / 分支 / PR | `git` |
| 准备说"搞定了" | `verification-before-completion` |
| 提交前自查 | `requesting-code-review` |
| 收到别人的 review | `receiving-code-review` |
| 实现完、准备合并 | `finishing-a-development-branch` |
| 2+ 个独立任务 | `dispatching-parallel-agents` |
| 需要隔离工作环境 | `using-git-worktrees` |
