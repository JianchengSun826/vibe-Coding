# requesting-code-review

> 在提交 PR 或声明完成之前，对自己的代码做系统性自查。

**类型**：流程类（Superpowers）— 不可跳过
**触发时机**：git commit / 创建 PR 之前

---

## 自查顺序

```
1. 正确性  ← 核心逻辑有没有 bug？
2. 安全性  ← 有没有安全漏洞？
3. 测试    ← 测试是否充分？
4. 可读性  ← 别人能看懂吗？
5. 性能    ← 有没有明显的性能问题？
```

---

## 各维度检查清单

### 1. 正确性

- [ ] 核心功能按预期工作？
- [ ] 边界条件处理了吗？（空值、0、最大值、负数）
- [ ] 错误处理完整吗？（数据库失败、网络超时、无效输入）
- [ ] 并发场景下有没有竞争条件？

**示例：发现自己的 bug**
```typescript
// 自查时发现：没有处理 items 为空数组的情况
function getFirstItem(items: string[]): string {
  return items[0].toUpperCase();  // ❌ items 为空时崩溃
}

// 修复后
function getFirstItem(items: string[]): string {
  if (items.length === 0) throw new Error('List is empty');
  return items[0].toUpperCase();
}
```

---

### 2. 安全性

- [ ] 用户输入有没有验证和清理？
- [ ] 有没有硬编码的密钥、密码、token？
- [ ] 权限检查完整吗？（任何用户都可以访问这个接口吗？）
- [ ] 错误信息有没有泄露内部实现细节？

**示例：发现安全问题**
```typescript
// 自查时发现：错误信息泄露了内部信息
if (!user) {
  throw new Error(`User with email ${email} not found in database`);
  // ❌ 攻击者可以用这个信息枚举有效邮箱
}

// 修复后
if (!user) {
  throw new UnauthorizedException('Invalid credentials');
  // ✓ 不区分"用户不存在"和"密码错误"
}
```

---

### 3. 测试

- [ ] 测试覆盖了正常路径吗？
- [ ] 测试覆盖了所有失败路径吗？
- [ ] 有没有测试边界条件？
- [ ] 测试之间有没有依赖关系（共享状态）？

**示例：发现测试不足**
```typescript
// 自查时发现：只测了登录成功，没测失败情况
describe('login', () => {
  test('成功 ✓', ...);
  // 缺少：
  // test('密码错误 → 401');
  // test('用户不存在 → 401');
  // test('账号被锁定 → 403');
});
```

---

### 4. 可读性

- [ ] 函数名和变量名清晰表达了意图？
- [ ] 复杂逻辑有没有注释？
- [ ] 函数是否专注于一件事？
- [ ] 有没有重复代码可以提取？

**示例：发现命名问题**
```typescript
// 自查时发现命名不清晰
const d = new Date();  // ❌ d 是什么？
const expiresAt = new Date();  // ✓

function p(u: User, t: string) { ... }  // ❌
function processUserToken(user: User, token: string) { ... }  // ✓
```

---

### 5. 性能

- [ ] 有没有 N+1 查询？
- [ ] 有没有在循环里做不必要的计算？
- [ ] 大数据集有没有分页？

**示例：发现 N+1 问题**
```typescript
// 自查时发现 N+1
const orders = await Order.findAll();
for (const order of orders) {
  order.user = await User.findById(order.userId);  // ❌ N 次查询
}

// 修复：用 JOIN 一次查询
const orders = await Order.findAll({ include: [User] });  // ✓
```

---

## 提交前检查命令

```bash
# 1. 查看本次改动
git diff HEAD

# 2. 确认没有调试代码遗留
grep -r "console.log\|debugger\|TODO\|FIXME" src/

# 3. 确认没有硬编码敏感信息
grep -r "password\|secret\|api_key" src/ --include="*.ts"

# 4. 跑完整测试
npm test

# 5. 跑 lint
npm run lint
```
