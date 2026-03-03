# git（Domain Skill）

> 提交规范、分支管理、PR 工作流、冲突解决。

**类型**：领域类 — 按需查阅
**触发时机**：git commit、创建分支、处理 PR、解决合并冲突

---

## Commit 规范（Conventional Commits）

格式：`type(scope): description`

| type | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(auth): add OAuth login` |
| `fix` | Bug 修复 | `fix(api): handle null user in login` |
| `refactor` | 重构（不改行为） | `refactor(auth): extract token utils` |
| `test` | 测试相关 | `test(auth): add login failure cases` |
| `docs` | 文档 | `docs: update API reference` |
| `chore` | 工程维护 | `chore: upgrade dependencies` |
| `perf` | 性能优化 | `perf(db): add index on users.email` |

**好 vs 坏 commit message**：

```bash
# ❌ 坏
git commit -m "fix bug"
git commit -m "update code"
git commit -m "wip"

# ✓ 好
git commit -m "fix(auth): handle database timeout in login endpoint

When database query times out, findUser returns undefined.
Added null check before accessing user.id to prevent 500 error.

Closes #123"
```

---

## 分支命名

```bash
feat/user-login          # 新功能
fix/login-500-error      # bug 修复
refactor/auth-module     # 重构
chore/upgrade-deps       # 维护工作
hotfix/payment-crash     # 紧急修复
```

---

## 常用工作流

### 开始新功能

```bash
git checkout main
git pull origin main          # 确保基于最新 main
git checkout -b feat/my-feature
# 开始工作...
```

### 提交前检查

```bash
git status                    # 查看修改了哪些文件
git diff                      # 查看具体改动
git diff --staged             # 查看已暂存的改动

# 有选择地暂存（不要 git add . 把不相关的文件都加进去）
git add src/auth/auth.service.ts
git add tests/auth/auth.service.spec.ts
```

### 创建 PR

```bash
git push origin feat/my-feature
gh pr create \
  --title "feat(auth): add user login" \
  --body "## 变更说明
实现了邮箱/密码登录，使用 JWT 返回 token。

## 测试方法
1. POST /auth/login
2. 验证返回 token

## Checklist
- [x] 测试通过
- [x] 无硬编码密钥"
```

---

## 解决合并冲突

```bash
git checkout main
git pull origin main
git checkout feat/my-feature
git rebase main               # 把 feature 的提交移到 main 的最新点

# 如果有冲突：
# 1. 打开冲突文件，手动解决（找 <<<, ===, >>> 标记）
# 2. git add <resolved-file>
# 3. git rebase --continue

# 验证解决结果
git log --oneline -5
npm test                      # 确认合并后测试仍然通过
```

---

## 常用补救操作

```bash
# 撤销最后一次 commit（保留改动）
git reset HEAD~1

# 修改最后一次 commit message
git commit --amend -m "新消息"

# 暂存当前工作（去做紧急任务）
git stash push -m "WIP: login form validation"
git stash pop                 # 取回

# 查看某个文件的历史
git log --oneline src/auth/auth.service.ts

# 查看某次提交的改动
git show abc1234
```

---

## Rebase vs Merge

| | Rebase | Merge |
|---|---|---|
| 历史 | 线性，干净 | 保留分支历史 |
| 适合 | feature branch 更新到最新 main | 合并 PR（保留记录） |
| 注意 | 不要 rebase 已经 push 的公共分支 | commit 历史会有 merge commit |
