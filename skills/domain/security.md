# security（Domain Skill）

> 认证、权限、输入验证、敏感数据处理。

**类型**：领域类 — 按需查阅
**触发时机**：写登录/注册、权限控制、处理用户输入、存储敏感数据时

---

## 密码存储

```typescript
// ❌ 绝对不能这样做
const user = { password: req.body.password };     // 明文存储
const user = { password: md5(req.body.password) }; // MD5/SHA1 容易破解

// ✓ 正确做法：使用 bcrypt 或 argon2
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;  // 推荐 10-14

// 注册时
const passwordHash = await bcrypt.hash(req.body.password, SALT_ROUNDS);
await db.create({ email, passwordHash });

// 登录时
const isMatch = await bcrypt.compare(req.body.password, user.passwordHash);
if (!isMatch) throw new UnauthorizedException('Invalid credentials');
```

---

## JWT 实现

```typescript
import jwt from 'jsonwebtoken';

// ❌ 错误做法
const token = jwt.sign({ userId }, 'my-secret');      // 硬编码 secret
const token = jwt.sign({ userId }, process.env.JWT_SECRET); // 没有过期时间

// ✓ 正确做法
const token = jwt.sign(
  { userId, email },
  process.env.JWT_SECRET!,   // 从环境变量读取
  { expiresIn: '7d' }        // 设置过期时间
);

// 验证 token
function verifyToken(token: string) {
  try {
    return jwt.verify(token, process.env.JWT_SECRET!);
  } catch (err) {
    throw new UnauthorizedException('Invalid or expired token');
  }
}
```

---

## 防止用户名枚举

```typescript
// ❌ 错误：区分了"用户不存在"和"密码错误"
if (!user) {
  return res.status(404).json({ error: 'User not found' });
}
if (!isPasswordMatch) {
  return res.status(401).json({ error: 'Wrong password' });
}
// 攻击者可以用这个判断哪些邮箱是有效账号

// ✓ 正确：统一返回相同的错误
if (!user || !isPasswordMatch) {
  return res.status(401).json({ error: 'Invalid credentials' });
}
```

---

## 输入验证

```typescript
import { z } from 'zod';  // 或者用 joi、class-validator

// 定义输入 schema（接受什么输入）
const LoginSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(128),
});

app.post('/auth/login', async (req, res) => {
  // 验证并解析（不通过会抛出错误）
  const { email, password } = LoginSchema.parse(req.body);
  // 之后的代码可以信任 email 和 password 是有效的
  ...
});

// ❌ 不要这样
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body;  // 直接使用，没有验证
  await userService.login(email, password);
});
```

---

## SQL 注入防护

```typescript
// ❌ 字符串拼接（SQL 注入）
const users = await db.query(
  `SELECT * FROM users WHERE email = '${email}'`
);
// 攻击者输入：' OR '1'='1 → 绕过认证

// ✓ 参数化查询
const users = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// ✓ 使用 ORM
const user = await User.findOne({ where: { email } });
```

---

## 权限控制

```typescript
// ❌ 只验证了登录，没有验证权限
app.delete('/admin/users/:id', authMiddleware, async (req, res) => {
  await userService.delete(req.params.id);  // 任何登录用户都能删用户！
});

// ✓ 分层验证：认证 + 授权
app.delete('/admin/users/:id',
  authMiddleware,              // Step 1: 验证 token（你是谁？）
  requireRole('admin'),        // Step 2: 验证角色（你有权限吗？）
  async (req, res) => {
    await userService.delete(req.params.id);
  }
);

// requireRole 中间件示例
function requireRole(role: string) {
  return (req, res, next) => {
    if (req.user.role !== role) {
      return res.status(403).json({ error: 'Forbidden' });
    }
    next();
  };
}
```

---

## 敏感数据不能出现在

```typescript
// ❌ 不能出现在以下地方
console.log('User logged in:', user);  // 可能打印 password_hash
git commit -m "fix" -a                 // 可能提交了 .env 文件
return res.json(user);                 // 返回了 password_hash 字段

// ✓ 返回用户数据时明确排除敏感字段
const { passwordHash, ...safeUser } = user;
return res.json(safeUser);
```

---

## OWASP Top 10 速查

| 风险 | 防护 |
|------|------|
| SQL 注入 | 参数化查询 / ORM |
| XSS | 输出时转义，CSP header |
| 敏感数据暴露 | HTTPS，不存明文密码 |
| 认证缺陷 | bcrypt 密码，JWT 有效期，失败次数限制 |
| 访问控制 | 每个端点都要验证权限 |
| 安全配置错误 | 关闭不必要的端口/功能，更新依赖 |
