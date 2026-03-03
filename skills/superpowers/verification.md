# verification-before-completion

> 在声明任务"完成"之前，必须真正运行验证命令并确认输出。

**类型**：流程类（Superpowers）— 不可跳过
**触发时机**：准备说"完成了"/"修好了"/"测试通过了"之前

---

## 核心原则

**证据先于断言**：不能说"我认为测试应该通过"，必须说"我跑了测试，输出是 PASS"。

---

## 需要验证的内容

| 要声明的 | 必须做的验证 |
|---------|------------|
| "功能实现完了" | `npm test` → 看到 PASS |
| "Bug 修好了" | 复现测试通过，且全部测试通过 |
| "性能优化好了" | 实际跑 Lighthouse / benchmark，对比数据 |
| "安全问题修了" | 跑安全扫描 / 手动验证攻击向量 |
| "PR 可以合并了" | CI 所有检查通过，无冲突 |

---

## 示例

### 示例 1：正确的验证流程

**场景**：实现了用户注册功能

```bash
# Step 1：跑单元测试
npm test src/auth/

# 必须看到：
# PASS src/auth/auth.service.spec.ts
# PASS src/auth/auth.controller.spec.ts
# Tests: 12 passed, 0 failed

# Step 2：跑集成测试
npm run test:integration

# Step 3：检查测试覆盖率
npm run test:coverage
# 确认覆盖率没有下降

# Step 4：手动验证核心路径
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"SecurePass123!"}'
# 期望：201 {"id": 1, "email": "test@example.com"}

# 然后才能说："注册功能实现完成"
```

---

### 示例 2：Bug Fix 验证

**场景**：修复了"登录接口当用户不存在时返回 500"

```bash
# Step 1：跑专门的复现测试
npm test -- --testNamePattern="用户不存在时应返回 401"
# 期望：PASS（之前这个测试是 FAIL 的，现在变绿了）

# Step 2：跑所有认证相关测试
npm test src/auth/
# 期望：全部 PASS，没有引入新 bug

# Step 3：验证修复没有破坏其他功能
npm test
# 期望：全部测试 PASS

# 然后才能说："Bug 已修复"
```

---

### 示例 3：性能优化验证

**场景**：优化了首页加载速度

```
# Step 1：优化前已经记录了基准数据
# LCP: 4.2s, TTI: 5.1s, Score: 45

# Step 2：实施优化后，再次跑 Lighthouse
npx lighthouse http://localhost:3000 --output json

# Step 3：对比数据
# LCP: 1.8s（从 4.2s → 1.8s，提升 57%）✓
# TTI: 2.3s（从 5.1s → 2.3s，提升 55%）✓
# Score: 89（从 45 → 89）✓

# 然后才能说："首页性能优化完成"
```

---

## 禁止的断言

```
❌ "我认为测试应该通过"
❌ "逻辑上看起来没问题"
❌ "我检查了代码，应该是对的"
❌ "之前的测试通过了，这个改动应该没问题"
❌ "本地没问题"（没有跑测试，只是手动点了一下）
```

## 与 TDD 的关系

TDD 在写代码时保证每步都有测试，verification 在最后做**全量验证**：

```
tdd：每个函数写完就验证（小循环）
verification：整个功能完成后验证（大循环）
```

两者不能互相替代。
