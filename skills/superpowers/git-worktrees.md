# using-git-worktrees

> 为每个独立任务创建一个隔离的工作目录，避免分支切换打断当前工作。

**类型**：流程类（Superpowers）
**触发时机**：需要同时在多个分支上工作，或需要隔离实验性改动

---

## 什么是 Git Worktree

普通工作方式：一个仓库 = 一个工作目录，切分支会影响当前文件。

Worktree：一个仓库可以有多个工作目录，每个目录对应一个分支。

```
repo/
├── .git/                      ← 共享的 git 历史
├── （主工作区，main 分支）
└── .worktrees/
    ├── feat-auth-refactor/    ← worktree 1，feat/auth-refactor 分支
    └── feat-home-perf/        ← worktree 2，feat/home-perf 分支
```

---

## 常用命令

```bash
# 创建新 worktree（同时创建新分支）
git worktree add .worktrees/feat-login -b feat/user-login

# 创建 worktree（使用已有分支）
git worktree add .worktrees/hotfix origin/hotfix-v2

# 查看所有 worktrees
git worktree list

# 删除 worktree（保留分支）
git worktree remove .worktrees/feat-login

# 删除 worktree + 分支
git worktree remove .worktrees/feat-login
git branch -d feat/user-login
```

---

## 实际使用场景

### 场景 1：紧急 hotfix

```
当前：正在 feat/new-dashboard 分支开发新功能（改了很多文件）

突然：生产环境报 bug，需要立刻修复

不用 worktree 的痛苦：
  - stash 所有改动（可能 stash 错）
  - 切到 main 分支
  - 修完切回来
  - unstash（可能有冲突）

用 worktree：
  git worktree add .worktrees/hotfix -b hotfix/login-500
  cd .worktrees/hotfix
  # 在这里修 bug，完全不影响 feat/new-dashboard 的工作区
  # 修完 commit + push + merge
  git worktree remove .worktrees/hotfix
  # 回到主工作区，一切如常
```

### 场景 2：并行功能开发

```
任务 A：重构认证模块（改 src/auth/）
任务 B：优化首页（改 src/pages/home/）

git worktree add .worktrees/auth-refactor -b feat/auth-refactor
git worktree add .worktrees/home-perf -b feat/home-perf

# Agent 1 在 .worktrees/auth-refactor/ 工作
# Agent 2 在 .worktrees/home-perf/ 工作
# 两个任务同时进行，互不干扰
```

### 场景 3：对比两个版本的行为

```
git worktree add .worktrees/old-version v1.2.0
git worktree add .worktrees/new-version v2.0.0

# 同时运行两个版本进行对比
cd .worktrees/old-version && npm start &   # 跑在 3000 端口
cd .worktrees/new-version && npm start &   # 跑在 3001 端口
```

---

## 注意事项

1. **不能在同一个分支上创建两个 worktree**：每个 worktree 必须对应不同分支
2. **`.worktrees/` 加入 `.gitignore`**：worktree 目录本身不应该被 git 追踪
3. **删除前确认已提交**：`git worktree remove` 前确认 worktree 里的改动已经 commit 或 push

```bash
# 推荐：在 .gitignore 里加上
.worktrees/
```
