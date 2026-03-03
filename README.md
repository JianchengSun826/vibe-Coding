# Vibe Coding 学习笔记

> 学习如何与 AI 高效协作进行软件开发

---

## 目录结构

```
vibe-Coding/
├── workflows/          # 工作流场景指南（什么情况下按什么顺序用哪些 skill）
├── skills/
│   ├── superpowers/    # 流程控制类 skill（控制「怎么做事」的纪律）
│   └── domain/         # 技术领域类 skill（具体技术参考手册）
├── quick-ref/          # 高频速查表
└── notes/              # 学习心得 / 实战记录
```

---

## 快速导航

### 工作流场景

| 文件 | 场景 |
|------|------|
| [完整工作流概览](workflows/00-overview.md) | 所有场景总结 |
| [场景 1：开发新功能](workflows/01-new-feature.md) | 从需求到 PR 的完整流程 |
| [场景 2：修复 Bug](workflows/02-bug-fix.md) | 系统诊断到修复验证 |
| [场景 3：代码审查](workflows/03-code-review.md) | 审查别人代码 / 处理收到的 review |
| [场景 4：大型任务并行](workflows/04-parallel-tasks.md) | 多个独立任务同时推进 |

---

### Superpowers — 流程控制类

> 这类 skill 控制「做事的纪律」，**不能跳过**。

| Skill | 一句话 | 触发时机 |
|-------|--------|---------|
| [brainstorming](skills/superpowers/brainstorming.md) | 实现前先探索需求方向 | 接到任何新功能需求 |
| [writing-plans](skills/superpowers/writing-plans.md) | 制定分步骤实现计划 | brainstorming 之后 |
| [tdd](skills/superpowers/tdd.md) | 先写测试再写代码 | 写任何功能代码之前 |
| [systematic-debugging](skills/superpowers/systematic-debugging.md) | 系统诊断根因，不猜测 | 遇到 bug / 测试失败 |
| [verification-before-completion](skills/superpowers/verification.md) | 说"完成"前强制跑验证 | 准备声明任务完成时 |
| [requesting-code-review](skills/superpowers/requesting-review.md) | 提交前自查代码质量 | commit / PR 之前 |
| [receiving-code-review](skills/superpowers/receiving-review.md) | 理性处理收到的 review | 收到他人代码审查意见 |
| [finishing-a-development-branch](skills/superpowers/finishing-branch.md) | 决定 merge / PR / 清理 | 实现完成、测试通过后 |
| [dispatching-parallel-agents](skills/superpowers/parallel-agents.md) | 启动多 agent 并行执行 | 有 2+ 个独立任务 |
| [using-git-worktrees](skills/superpowers/git-worktrees.md) | 创建隔离工作环境 | 需要同时处理多个互不影响的任务 |

---

### Domain — 技术领域类

> 这类 skill 是「技术参考手册」，**按需随时查阅**。

| Skill | 触发场景 | 典型问题 |
|-------|---------|---------|
| [git](skills/domain/git.md) | 提交、分支、PR、合并冲突 | commit 规范、rebase vs merge |
| [review](skills/domain/review.md) | 审查代码质量 | 按什么顺序审查、关注什么 |
| [test](skills/domain/test.md) | 写测试 / 测试挂了 / mock 策略 | 单元 vs 集成 vs e2e |
| [db](skills/domain/db.md) | SQL / 表设计 / 慢查询 | 索引、N+1 问题、迁移 |
| [perf](skills/domain/perf.md) | 性能优化 | 先 profiling，再优化 |
| [security](skills/domain/security.md) | 认证 / 权限 / 输入验证 | JWT、密码存储、OWASP |

---

### 速查

- [Skill 触发速查表](quick-ref/skill-trigger-table.md) — 「什么情况用哪个 skill」一页汇总

---

## 核心原则

**Superpowers skill 控制流程纪律（不能跳过），Domain skill 是随时查阅的技术手册（按需调用）。**

两者配合：superpowers 告诉你「先写测试再实现」，`test` skill 告诉你「测试怎么写好」。
