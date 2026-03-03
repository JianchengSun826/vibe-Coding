# 场景 2：修复 Bug

> 用户说："登录接口偶尔返回 500"

---

## 完整流程

```
1. systematic-debugging       ← 诊断根因，不猜
2. tdd                        ← 写一个复现 bug 的失败测试
3. 修复代码
   ↑ 过程中按需:
     db                       ← 如果是慢查询 / 数据库锁
     perf                     ← 如果是性能瓶颈
4. verification               ← 确认测试通过，不是侥幸
5. git                        ← commit fix，写清楚 fix 了什么
```

---

## 各步骤说明

### Step 1 — `/systematic-debugging`

**目的**：通过证据找根因，不靠猜测。

**流程**：
```
1. 复现问题（能稳定复现才能修）
2. 收集证据（日志、错误信息、堆栈追踪）
3. 提出假设（基于证据，不是直觉）
4. 验证假设（修改一个变量，观察结果）
5. 找到根因 → 修复
```

**示例**：

问题：`POST /auth/login` 偶尔返回 500

```
Step 1 复现：在压测下每 50 个请求出现 1 次
Step 2 收集证据：
  - 查看服务器日志：TypeError: Cannot read property 'id' of undefined
  - 堆栈指向：auth.service.ts:47
  - Line 47：const userId = user.id;

Step 3 假设：
  - 假设：某些情况下 findUserByEmail 返回 null，但没有做 null 检查
  - 根据：错误是"Cannot read property of undefined"

Step 4 验证假设：
  - 查看 findUserByEmail 的实现 → 确认：数据库查询超时时返回 undefined

根因：数据库偶尔超时，findUserByEmail 返回 undefined，
      但调用方没有 null 检查，直接访问 .id 导致崩溃。
```

**错误做法**（不要猜）：
```
❌ "可能是并发问题，我加个锁试试"
❌ "可能是数据库连接池不够，我改大一点"
❌ "可能是内存溢出，重启服务器"
```

---

### Step 2 — `/tdd`（写复现测试）

**目的**：在修复之前，先写一个能复现 bug 的失败测试。这样修复后测试变绿，就证明真的修好了。

```typescript
// 复现测试：数据库查询返回 null 时，不应该 500
test('当用户不存在时，login 应返回 401 而不是抛出异常', async () => {
  // mock 数据库返回 null（模拟查询不到用户）
  jest.spyOn(userRepo, 'findByEmail').mockResolvedValue(null);

  const response = await request(app)
    .post('/auth/login')
    .send({ email: 'nobody@example.com', password: '123456' });

  expect(response.status).toBe(401);  // 应该是 401，不是 500
  // 当前这个测试会失败（因为 bug 还没修）
});
```

---

### Step 3 — 修复代码

根据根因修复：

```typescript
// 修复前（有 bug）
async login(email: string, password: string) {
  const user = await userRepo.findByEmail(email);
  const userId = user.id;  // ❌ 如果 user 是 null，直接崩溃
  ...
}

// 修复后
async login(email: string, password: string) {
  const user = await userRepo.findByEmail(email);
  if (!user) {
    throw new UnauthorizedException('Invalid credentials');  // ✓ 处理 null
  }
  const userId = user.id;
  ...
}
```

---

### Step 3 中途 — `/db`（如果是数据库问题）

如果根因是数据库超时，DB skill 会指导：
- 检查慢查询日志（`EXPLAIN ANALYZE` 查看查询计划）
- 是否缺少索引（`email` 字段有没有索引？）
- 连接池配置是否合理（连接数上限太低会导致等待超时）

---

### Step 4 — `/verification-before-completion`

```bash
# 跑刚才写的复现测试，确认它现在通过了
npm test auth.service.spec.ts

# 跑所有测试，确认没有引入新 bug
npm test

# 输出必须是：
# PASS src/auth/auth.service.spec.ts
# Tests: 1 passed (包含新加的复现测试)
```

---

### Step 5 — `/git`

```bash
git add src/auth/auth.service.ts tests/auth/auth.service.spec.ts
git commit -m "fix(auth): handle null user when database query returns undefined

Fixes 500 error on POST /auth/login when database times out.
Added null check before accessing user.id.
Added regression test to prevent recurrence."
```

Fix commit 要说明：
1. 修了什么
2. 为什么会出现（根因）
3. 怎么修的（可选）
