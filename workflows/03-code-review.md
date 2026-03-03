# 场景 3：代码审查

两种子场景：**审查别人的代码** 和 **收到别人对你代码的反馈**。

---

## 子场景 A：审查别人的代码

> 你是 reviewer，收到了一个 PR

### 流程

```
1. review       ← 按顺序审查：正确性 → 安全 → 测试 → 可维护性
2. security     ← 专门检查认证、输入验证、权限相关代码
```

### `/review` — 审查顺序

**第 1 步：正确性（最重要）**

- 核心逻辑有没有 bug？
- 边界条件处理了吗？
- 错误处理完整吗？

示例——发现逻辑错误：
```typescript
// PR 中的代码
function calculateDiscount(price: number, userType: string) {
  if (userType === 'vip') {
    return price * 0.8;
  }
  return price * 0.9;  // ❌ 非 VIP 用户也打了 9 折？业务逻辑错误
}
// Review 评论："非 VIP 用户应该全价，这里似乎有误。请确认业务需求。"
```

**第 2 步：安全**（见下方 security）

**第 3 步：测试**

- 测试覆盖了核心路径吗？
- 有没有测试边界条件（空值、最大值、并发）？
- 测试是否真的在测业务逻辑，而不是在测 mock？

示例——发现测试不足：
```typescript
// PR 的测试只测了正常路径
test('login success', ...) ✓

// 缺少的测试
// ❌ 没有测试：密码错误的情况
// ❌ 没有测试：用户不存在的情况
// ❌ 没有测试：账号被锁定的情况
// Review 评论："建议补充失败路径的测试，尤其是认证相关代码需要测试所有失败场景。"
```

**第 4 步：可维护性**

- 函数是否做了超过一件事？
- 命名是否清晰？
- 复杂逻辑有没有注释？

示例——发现可维护性问题：
```typescript
// PR 中的代码
function p(u: any, t: string) {  // ❌ 命名无意义
  return t === 'a' ? u.n : u.e;  // ❌ 完全看不懂在做什么
}
// Review 评论："函数名和参数名不够清晰，建议重命名并添加注释说明用途。"
```

---

### `/security`（专门检查安全问题）

**触发时机**：PR 涉及认证、权限、用户输入时。

检查清单：

**输入验证**：
```typescript
// ❌ 直接使用用户输入（SQL 注入风险）
const user = await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✓ 参数化查询
const user = await db.query('SELECT * FROM users WHERE email = $1', [email]);
// Review 评论："这里存在 SQL 注入风险，请改用参数化查询。"
```

**权限检查**：
```typescript
// ❌ 只验证了登录，没有验证权限
app.delete('/admin/users/:id', authMiddleware, async (req, res) => {
  await userService.delete(req.params.id);  // ❌ 任何登录用户都能删除用户
});

// ✓ 验证角色
app.delete('/admin/users/:id', authMiddleware, requireRole('admin'), async (req, res) => {
  ...
});
// Review 评论："这个端点需要管理员权限校验，当前所有登录用户都可以访问。"
```

---

## 子场景 B：收到对你代码的反馈

> 有人 review 了你的代码，给了你建议

### 流程

```
receiving-code-review  ← 避免两种极端：盲目接受 or 防御性拒绝
```

### `/receiving-code-review`

**核心原则**：收到反馈时，先理解，再验证，再决策。

**三种处理方式**：

**1. 理解后接受（技术上正确）**

```
Reviewer: "这里的 N+1 查询会造成性能问题，建议用 JOIN 替代。"

处理：
  - 先理解什么是 N+1 问题（查 /db skill）
  - 确认：是的，在循环里调用数据库确实是 N+1
  - 接受并修改
```

**2. 理解后讨论（技术上有分歧）**

```
Reviewer: "这个函数太长了，应该拆成 5 个小函数。"

处理：
  - 理解 reviewer 的关切（可维护性）
  - 思考：拆成 5 个是否过度分散，增加认知负担？
  - 回复："我理解你对函数长度的顾虑，
          但这 5 个步骤是一个原子操作，
          拆分后调用者需要了解内部顺序，反而更难维护。
          我可以添加注释来提高可读性，你觉得这样可以吗？"
```

**3. 不盲目接受（reviewer 可能有误）**

```
Reviewer: "用 any 类型就行了，不需要定义这么复杂的类型。"

处理：
  - 不要因为 reviewer 说了就改
  - 评估：any 会丢失类型安全，是有代价的
  - 回复："我理解 any 更简单，但这里的类型定义
          可以让编译器帮我们捕获错误。
          你是否有具体的场景觉得当前类型太复杂？"
```

**避免的行为**：
- ❌ "好的好的，我马上改" — 没有理解就接受
- ❌ "你不懂我的代码" — 防御性拒绝，关闭讨论
- ✓ "我理解你的意思，让我看一下..." — 先理解，再决策
