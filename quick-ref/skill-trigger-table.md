# Skill 触发速查表

> 打开即用。遇到对应情况，直接用 `/skill名` 触发。

---

## 按情况查

| 情况 | 用哪个 | 命令 |
|------|--------|------|
| 接到新功能需求，准备动手前 | brainstorming → writing-plans | `/brainstorming` |
| 写任何功能代码之前 | tdd | `/tdd` |
| 遇到 bug / 测试失败 | systematic-debugging | `/systematic-debugging` |
| 写认证、权限、输入验证 | security | `/security` |
| 写 SQL / 设计表 / 慢查询 | db | `/db` |
| 写测试 / 测试挂了 / 不知道怎么 mock | test | `/test` |
| 性能问题（接口慢、页面慢） | perf | `/perf` |
| 提交 / 创建分支 / PR / 合并冲突 | git | `/git` |
| 准备说"搞定了" / "修好了" | verification-before-completion | `/verification-before-completion` |
| commit / PR 之前自查代码 | requesting-code-review | `/requesting-code-review` |
| 收到别人对你代码的 review | receiving-code-review | `/receiving-code-review` |
| 审查别人的 PR | review + security | `/review` |
| 实现完成，准备合并 | finishing-a-development-branch | `/finishing-a-development-branch` |
| 有 2+ 个独立任务 | dispatching-parallel-agents | `/dispatching-parallel-agents` |
| 需要隔离工作环境 | using-git-worktrees | `/using-git-worktrees` |

---

## 按场景查

### 开发新功能

```
/brainstorming → /writing-plans → /tdd
  + /security（认证相关）
  + /db（数据库相关）
  + /test（不确定怎么写测试）
→ /verification-before-completion
→ /requesting-code-review
→ /git
→ /finishing-a-development-branch
```

### 修复 Bug

```
/systematic-debugging → /tdd（写复现测试）→ 修复
  + /db（数据库问题）
  + /perf（性能问题）
→ /verification-before-completion
→ /git
```

### 代码审查

```
审查别人: /review → /security
收到反馈: /receiving-code-review
```

### 大型并行任务

```
/brainstorming → /writing-plans
→ /using-git-worktrees（隔离）
→ /dispatching-parallel-agents（并行）
→ /verification-before-completion + /requesting-code-review + /finishing-a-development-branch
```

---

## 核心原则

**Superpowers skill（流程类）= 纪律，不能跳过**
- brainstorming, writing-plans, tdd, systematic-debugging, verification, requesting-review, receiving-review, finishing-branch, parallel-agents, git-worktrees

**Domain skill（领域类）= 手册，按需查阅**
- git, review, test, db, perf, security

两者分工：superpowers 告诉你**什么时候**做**什么**，domain 告诉你**怎么做好**。
