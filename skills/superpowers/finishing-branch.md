# finishing-a-development-branch

> 实现完成、测试通过后，决定如何整合代码（merge、PR、squash、清理）。

**类型**：流程类（Superpowers）— 不可跳过
**触发时机**：功能实现完成，准备合并到主分支

---

## 决策树

```
实现完成 + 测试全通过
  ↓
这个改动需要他人 review 吗？
  ├── 是 → 创建 PR → 请求 review → merge
  └── 否 → 直接 merge 到 main
          ↓
      commits 要整理吗？
        ├── 是（提交历史凌乱）→ squash merge
        └── 否（提交历史清晰）→ merge commit 或 rebase
```

---

## 各种 merge 方式

### 1. 创建 PR（推荐用于团队协作）

```bash
git push origin feat/user-login
gh pr create \
  --title "feat(auth): add email/password login with JWT" \
  --body "## What
Implements user login with email/password authentication.
Returns JWT token on success.

## How to test
1. POST /auth/register with valid email/password
2. POST /auth/login with same credentials
3. Verify JWT token in response

## Checklist
- [x] Unit tests pass
- [x] Integration tests pass
- [x] No hardcoded secrets
- [x] Error handling complete"
```

---

### 2. Squash Merge（整理提交历史）

**适用**：功能分支上有很多 WIP commit（"fix typo"、"debug"、"try this"）

```bash
# 在 GitHub PR 中选择 "Squash and merge"
# 或者命令行：
git checkout main
git merge --squash feat/user-login
git commit -m "feat(auth): add email/password login with JWT"
```

**效果**：把整个 feature branch 的所有提交合并成一个干净的提交。

---

### 3. Rebase + Merge（保持线性历史）

```bash
git checkout feat/user-login
git rebase main        # 把 feature branch 的提交移到 main 的最新点
git checkout main
git merge feat/user-login --ff-only
```

---

### 4. 直接 Merge Commit

```bash
git checkout main
git merge feat/user-login
git push origin main
```

---

## 合并后的清理

```bash
# 删除本地 feature branch
git branch -d feat/user-login

# 删除远程 feature branch
git push origin --delete feat/user-login

# 更新本地 main
git pull origin main
```

---

## Worktree 清理

如果用了 git worktrees（见 [using-git-worktrees](./git-worktrees.md)）：

```bash
# 查看所有 worktrees
git worktree list

# 删除 worktree
git worktree remove .worktrees/feat-user-login

# 删除分支
git branch -d feat/user-login
```

---

## 示例：完整的收尾流程

```bash
# 1. 确认最终状态
git status          # 干净的工作区
npm test            # 全部测试通过

# 2. 整理提交（可选）
git log --oneline feat/user-login  # 查看提交历史
# 如果有很多 "fix"、"wip" 提交，考虑 squash

# 3. 创建 PR
git push origin feat/user-login
gh pr create --title "feat(auth): add login" --body "..."

# 4. 等待 review + CI 通过

# 5. Merge

# 6. 清理
git branch -d feat/user-login
git push origin --delete feat/user-login
```
