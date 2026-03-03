# 场景 1：开发新功能

> 用户说："帮我实现用户登录功能"

---

## 完整流程

```
1. brainstorming              ← 探索需求，避免做错方向
2. writing-plans              ← 制定多步骤实现计划
3. tdd                        ← 先写测试，再写代码
   ↑ 过程中随时调用:
     security                 ← 处理认证/密码/token 时
     db                       ← 设计 users 表、写查询时
     test                     ← 不确定如何组织测试时
4. verification               ← 声明"完成"之前强制验证
5. requesting-review          ← 提交前自我审查代码质量
6. git                        ← 提交规范、PR 前检查
7. finishing-branch           ← 决定 merge / PR / 清理
```

---

## 各步骤说明

### Step 1 — `/brainstorming`

**目的**：在动手前弄清楚「要做什么」和「怎么做最合理」。

AI 会问你：
- 登录方式是邮箱/密码，还是 OAuth（Google、GitHub）？
- Session cookie 还是 JWT token？
- 需要「记住我」功能吗？Token 有效期是多久？
- 是否需要多设备同时登录？
- 已有用户表了吗？结构是什么？

**没有 brainstorming 的后果**：
直接开始写代码，实现了邮箱登录，然后才发现需求是 Google OAuth。白做。

---

### Step 2 — `/writing-plans`

**目的**：把需求拆分成有序的子任务，识别依赖关系。

示例计划（登录功能）：
```
1. 设计 users 表结构（email, password_hash, created_at）
2. 实现密码哈希工具函数（使用 bcrypt）
3. 实现 POST /auth/register 接口
4. 实现 POST /auth/login 接口，返回 JWT
5. 实现 JWT 验证中间件
6. 为 step 3-5 编写集成测试
```

**计划的价值**：先写步骤 1（数据库），才能写步骤 2（业务逻辑），顺序清晰。

---

### Step 3 — `/tdd`

**目的**：先写一个「失败的测试」，再写让它通过的最小代码。

红绿循环：
```
写测试（Red）→ 跑测试（失败）→ 写最小实现（Green）→ 重构 → 下一个测试
```

示例：
```typescript
// 先写测试（此时 hashPassword 函数还不存在）
test('hashPassword 应该返回与原密码不同的字符串', async () => {
  const hash = await hashPassword('mypassword123');
  expect(hash).not.toBe('mypassword123');
  expect(hash.length).toBeGreaterThan(20);
});

// 跑测试 → 失败（函数不存在）
// 然后才写实现
export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, 10);
}
// 跑测试 → 通过 ✓
```

---

### Step 3 中途 — `/security`（处理认证相关代码时）

触发时机：写密码存储、JWT 生成、session 管理时。

Security skill 会提醒你：
- ✓ 用 bcrypt/argon2，不能用 MD5/SHA1 存密码
- ✓ JWT secret 必须从环境变量读取，不能硬编码
- ✓ 登录失败不能区分「用户不存在」和「密码错误」（防枚举）
- ✓ 设置合理的 token 过期时间

---

### Step 3 中途 — `/db`（设计表结构时）

触发时机：写 CREATE TABLE、设计索引、写登录查询时。

DB skill 会提醒你：
- ✓ email 字段需要唯一索引
- ✓ 查询：`SELECT * FROM users WHERE email = ?`（参数化，防 SQL 注入）
- ✓ 不要在 users 表存明文密码

---

### Step 4 — `/verification-before-completion`

**目的**：在说"完成了"之前，必须真正跑验证命令并确认输出。

不接受：
- "我认为测试应该通过"
- "逻辑上看起来没问题"

必须做：
```bash
npm test                    # 跑所有测试，看到 PASS
npm run test:coverage       # 检查覆盖率
curl -X POST /auth/login    # 手动测试接口
```

---

### Step 5 — `/requesting-code-review`

提交前自查清单：
- [ ] 所有测试通过？
- [ ] 有没有硬编码的密钥或敏感信息？
- [ ] 错误处理是否完整（登录失败返回 401，不泄露细节）？
- [ ] 是否有遗漏的边界情况（空密码、超长邮箱）？

---

### Step 6 — `/git`

```bash
git add src/auth/ tests/auth/
git commit -m "feat(auth): add email/password login with JWT"
```

Commit 规范：`type(scope): description`

---

### Step 7 — `/finishing-a-development-branch`

AI 会询问你：
- 直接 merge 到 main？
- 创建 PR 并请求 review？
- 需要 squash commits？
- 清理 feature branch？
