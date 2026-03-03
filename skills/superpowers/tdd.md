# tdd（Test-Driven Development）

> 先写一个失败的测试，再写让它通过的最小代码，然后重构。

**类型**：流程类（Superpowers）— 不可跳过
**触发时机**：写任何功能代码之前

---

## 核心循环

```
Red   → 写一个失败的测试（测试先于实现）
Green → 写最小代码让测试通过（不过度实现）
Refactor → 在测试保护下改善代码质量
↓
重复
```

---

## 为什么先写测试

1. **强迫你想清楚接口**：在写实现之前，先想「这个函数接收什么，返回什么，怎么用」
2. **测试是活文档**：看测试比看代码更容易理解「这个函数应该怎么用」
3. **防止过度实现**：只实现让测试通过所需的最小代码
4. **重构有安全网**：随时可以放心重构，测试会告诉你是否破坏了功能

---

## 示例

### 示例 1：工具函数

**需求**：实现一个验证邮箱格式的函数

**Step 1：先写测试（Red）**
```typescript
// email.utils.spec.ts
import { isValidEmail } from './email.utils';  // 这个文件还不存在！

describe('isValidEmail', () => {
  test('有效邮箱返回 true', () => {
    expect(isValidEmail('user@example.com')).toBe(true);
    expect(isValidEmail('user+tag@sub.domain.com')).toBe(true);
  });

  test('无效邮箱返回 false', () => {
    expect(isValidEmail('')).toBe(false);
    expect(isValidEmail('not-an-email')).toBe(false);
    expect(isValidEmail('@no-user.com')).toBe(false);
    expect(isValidEmail('no-at-sign.com')).toBe(false);
  });
});

// 跑测试 → 失败（文件不存在）✓ 这是预期的
```

**Step 2：写最小实现（Green）**
```typescript
// email.utils.ts
export function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

// 跑测试 → 通过 ✓
```

**Step 3：重构（Refactor）**
```typescript
// 测试通过后，可以考虑是否需要更严格的验证
// 但不要在没有测试驱动的情况下"预防性地"加功能
```

---

### 示例 2：API 接口

**需求**：`POST /users` 创建用户

**先写集成测试**：
```typescript
// users.spec.ts
describe('POST /users', () => {
  test('有效数据 → 返回 201 和用户对象', async () => {
    const response = await request(app)
      .post('/users')
      .send({ email: 'test@example.com', password: 'SecurePass123!' });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id: expect.any(Number),
      email: 'test@example.com',
      // 注意：不应该返回 password 或 password_hash
    });
    expect(response.body.password).toBeUndefined();
  });

  test('重复邮箱 → 返回 409', async () => {
    // 先创建一个用户
    await createUser('existing@example.com');

    const response = await request(app)
      .post('/users')
      .send({ email: 'existing@example.com', password: 'Pass123!' });

    expect(response.status).toBe(409);
  });

  test('无效邮箱格式 → 返回 400', async () => {
    const response = await request(app)
      .post('/users')
      .send({ email: 'not-an-email', password: 'Pass123!' });

    expect(response.status).toBe(400);
  });
});

// 先跑 → 全部失败（接口还没实现）
// 然后实现接口，让测试一个个变绿
```

---

### 示例 3：Bug 复现测试

**场景**：登录接口当数据库返回 null 时崩溃

```typescript
test('用户不存在时应返回 401，不应抛出异常', async () => {
  // 模拟数据库返回 null
  jest.spyOn(userRepo, 'findByEmail').mockResolvedValue(null);

  const response = await request(app)
    .post('/auth/login')
    .send({ email: 'ghost@example.com', password: '123' });

  // 当前这个测试会失败（bug 会导致 500）
  // 修复 bug 后这个测试变绿，并永久防止回归
  expect(response.status).toBe(401);
});
```

---

## 常见误区

| 错误做法 | 正确做法 |
|---------|---------|
| 先写代码，再补测试 | 先写测试，再写代码 |
| 测试只覆盖正常路径 | 测试所有路径（正常 + 错误 + 边界） |
| 测试测的是实现细节 | 测试测的是行为（输入→输出） |
| 一次写很多测试再实现 | 一次一个测试，Red → Green → Refactor |

---

## 与其他 skill 的配合

- 不知道测试怎么写 → 调用 `/test`（Domain skill）
- 测试涉及数据库操作 → 调用 `/db`
- 测试涉及认证逻辑 → 调用 `/security`
