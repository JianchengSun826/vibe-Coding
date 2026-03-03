# test（Domain Skill）

> 测试类型选择、测试组织、mock 策略、解决 flaky test。

**类型**：领域类 — 按需查阅
**触发时机**：写测试、测试挂了、不知道怎么 mock、测试不稳定

---

## 三种测试类型

| 类型 | 测什么 | 速度 | 数量 |
|------|--------|------|------|
| 单元测试 | 单个函数/类 | 极快（ms） | 最多 |
| 集成测试 | 模块间交互 | 中等（秒） | 适量 |
| E2E 测试 | 完整用户流程 | 慢（分钟） | 少量 |

**测试金字塔**：单元测试最多，E2E 测试最少。

---

## 单元测试

测单个函数，隔离所有外部依赖（用 mock 替代）。

```typescript
// 被测函数
export function calculateTotal(items: CartItem[], discount: number): number {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.qty, 0);
  return subtotal * (1 - discount);
}

// 单元测试
describe('calculateTotal', () => {
  test('计算总价（含折扣）', () => {
    const items = [
      { price: 100, qty: 2 },
      { price: 50, qty: 1 },
    ];
    expect(calculateTotal(items, 0.1)).toBe(225);  // 250 * 0.9
  });

  test('空购物车返回 0', () => {
    expect(calculateTotal([], 0)).toBe(0);
  });

  test('0 折扣返回原价', () => {
    expect(calculateTotal([{ price: 100, qty: 1 }], 0)).toBe(100);
  });
});
```

---

## 集成测试

测试真实的 HTTP 请求 + 数据库（或 mock 数据库）。

```typescript
// 使用 supertest 测试 API 接口
import request from 'supertest';
import app from '../app';

describe('POST /auth/login', () => {
  beforeEach(async () => {
    // 每个测试前准备数据
    await db.user.create({
      email: 'test@example.com',
      passwordHash: await bcrypt.hash('correct-password', 10),
    });
  });

  afterEach(async () => {
    // 每个测试后清理数据（避免污染其他测试）
    await db.user.deleteMany({ where: { email: 'test@example.com' } });
  });

  test('正确密码 → 200 + token', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'correct-password' });

    expect(res.status).toBe(200);
    expect(res.body.token).toBeDefined();
  });

  test('错误密码 → 401', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'wrong-password' });

    expect(res.status).toBe(401);
    expect(res.body.token).toBeUndefined();
  });
});
```

---

## Mock 策略

### 什么时候 mock

- ✓ 外部服务（数据库、第三方 API、邮件服务）→ 单元测试时 mock
- ✗ 不要 mock 被测代码本身的逻辑

### Mock 示例

```typescript
// mock 一个模块
jest.mock('../services/email.service', () => ({
  sendWelcomeEmail: jest.fn().mockResolvedValue({ success: true }),
}));

// mock 一个函数
const mockFindUser = jest.spyOn(userRepo, 'findByEmail')
  .mockResolvedValue(null);  // 模拟用户不存在

// 验证 mock 被调用
expect(mockFindUser).toHaveBeenCalledWith('test@example.com');
expect(mockFindUser).toHaveBeenCalledTimes(1);

// 每个测试后重置 mock
afterEach(() => {
  jest.clearAllMocks();
});
```

### 模拟不同场景

```typescript
// 模拟成功
mockFn.mockResolvedValue({ id: 1, email: 'test@example.com' });

// 模拟失败
mockFn.mockRejectedValue(new Error('Database connection failed'));

// 第一次成功，第二次失败
mockFn
  .mockResolvedValueOnce({ id: 1 })
  .mockRejectedValueOnce(new Error('Timeout'));
```

---

## 测试组织

```
tests/
├── unit/                    # 单元测试（纯函数）
│   └── utils/
│       └── email.utils.spec.ts
├── integration/             # 集成测试（API + 数据库）
│   └── auth/
│       └── login.spec.ts
└── e2e/                     # E2E 测试（完整用户流程）
    └── user-registration.spec.ts
```

或者放在被测代码旁边（更常见）：
```
src/
├── auth/
│   ├── auth.service.ts
│   ├── auth.service.spec.ts     ← 单元测试放在旁边
│   └── auth.controller.spec.ts
```

---

## 解决 Flaky Test（不稳定测试）

常见原因和解决方法：

| 原因 | 解决方法 |
|------|---------|
| 测试间共享数据库状态 | `afterEach` 清理数据 |
| 依赖时间（`new Date()`）| mock 时间：`jest.useFakeTimers()` |
| 异步操作未等待 | 确保 `await` 了所有异步调用 |
| 依赖外部 API | mock 掉外部请求 |
| 测试顺序依赖 | 每个测试应该独立，不依赖前一个测试的状态 |

```typescript
// ❌ 时间相关的 flaky test
test('token 应该在 1 小时后过期', () => {
  const token = generateToken(user);
  // 1 小时后验证... 不可能等 1 小时
});

// ✓ mock 时间
test('token 应该在 1 小时后过期', () => {
  jest.useFakeTimers();
  const token = generateToken(user);

  jest.advanceTimersByTime(60 * 60 * 1000 + 1);  // 前进 1 小时 + 1ms

  expect(() => verifyToken(token)).toThrow('Token expired');
  jest.useRealTimers();
});
```

---

## 好的测试命名

```typescript
// ❌ 坏：描述实现
test('调用 findByEmail 并返回 user 对象')

// ✓ 好：描述行为（given/when/then 格式）
test('当用户存在时，登录应该返回 JWT token')
test('当密码错误时，登录应该返回 401')
test('当账号不存在时，注册应该创建新用户并返回 201')
```
