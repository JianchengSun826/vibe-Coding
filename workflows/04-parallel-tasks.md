# 场景 4：大型任务并行处理

> 用户说："重构认证模块，同时优化首页性能"

---

## 完整流程

```
1. brainstorming                    ← 厘清两个任务的边界，确认它们真的独立
2. writing-plans                    ← 把每个任务拆成子任务
3. using-git-worktrees              ← 两个任务各自独立的工作目录
4. dispatching-parallel-agents      ← 并行执行（两个 AI agent 同时工作）
   或 subagent-driven-development   ← 在当前 session 内顺序/并行执行
   ↑ 过程中:
     perf     → 首页性能任务
     security / db  → 认证模块任务
5. verification + requesting-review + finishing-branch
```

---

## 关键前提：任务必须真正独立

**可以并行的条件**（都满足才行）：
- ✓ 修改的文件不重叠
- ✓ 不共享运行时状态（同一数据库表、同一缓存 key）
- ✓ 一个任务的完成不依赖另一个任务的结果

**示例判断**：

| 任务组合 | 能并行? | 原因 |
|---------|--------|------|
| 重构 auth 模块 + 优化首页性能 | ✓ 能 | 文件不重叠，逻辑独立 |
| 修改 users 表 + 修改 orders 表 | ✓ 能 | 数据库表不同 |
| 修改用户注册逻辑 + 修改用户登录逻辑 | ⚠️ 谨慎 | 共享 users 表，可能冲突 |
| 前端改登录页 + 后端改登录接口 | ✓ 能 | 分层独立，接口契约提前约定好 |

---

## Step 1 — `/brainstorming`

**目的**：确认任务边界，发现潜在冲突。

AI 会问：
- 这两个任务会碰同一个文件吗？（`auth/user.service.ts` vs `home/home.component.tsx`）
- 认证模块重构后，接口签名会变吗？首页有没有用到认证相关的 API？
- 数据库有没有共享的表？

---

## Step 2 — `/writing-plans`

为每个任务单独制定计划：

**任务 A：重构认证模块**
```
A1. 审查现有认证代码，识别问题
A2. 提取 JWT 工具函数到独立模块
A3. 统一错误处理（返回标准格式）
A4. 补充缺失的单元测试
A5. 更新 API 文档
```

**任务 B：优化首页性能**
```
B1. Lighthouse 跑分，定位瓶颈
B2. 优化图片懒加载
B3. 拆分首屏 JS Bundle
B4. 添加 HTTP 缓存头
B5. 验证优化效果（对比跑分）
```

---

## Step 3 — `/using-git-worktrees`

为每个任务创建独立工作目录，避免互相干扰：

```bash
# 任务 A 的 worktree（认证重构）
git worktree add .worktrees/auth-refactor -b feat/auth-refactor

# 任务 B 的 worktree（首页优化）
git worktree add .worktrees/home-perf -b feat/home-perf
```

**好处**：
- 两个任务可以真正同时进行，不会互相 stash
- 每个任务有自己干净的 working tree
- 出问题只影响一个 worktree，不污染主分支

---

## Step 4A — `/dispatching-parallel-agents`（推荐用于大任务）

让两个 AI agent 同时工作：

```
Agent 1 → feat/auth-refactor worktree
  - 执行认证模块重构计划 A1~A5
  - 随时调用 security、db skill

Agent 2 → feat/home-perf worktree
  - 执行首页优化计划 B1~B5
  - 随时调用 perf skill
```

两个 agent 完全并行，互不干扰。

---

## Step 4B — `/subagent-driven-development`（用于当前 session 内）

如果不想派发给外部 agent，在当前 session 内有序推进：

```
主 agent 负责协调：
  1. 先完成独立的、不影响其他任务的子任务
  2. 识别阻塞关系，优先解锁阻塞项
  3. 在子任务间切换时保持上下文清晰
```

---

## Step 5 — 收尾三件套

每个任务分别走：
```
verification              ← 各自的测试通过
requesting-code-review    ← 各自自查
finishing-branch          ← 分别 PR 或 merge
```

**注意**：如果两个分支都要 merge 到 main，先合并一个，再 rebase 另一个到最新 main，避免冲突。
