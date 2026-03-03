# writing-plans

> 把需求拆分成有序的、可执行的子任务，并识别依赖关系。

**类型**：流程类（Superpowers）— 不可跳过（多步骤任务）
**触发时机**：brainstorming 之后，任务包含 3+ 个步骤时

---

## 为什么需要它

没有计划直接开始写代码，常见问题：
- 写到一半发现依赖了还没实现的模块
- 做了 step 3 才发现 step 1 的设计不对，需要推翻重来
- 多个子任务顺序混乱，互相阻塞

有了计划：依赖关系清晰，每步都有明确的完成标准。

---

## 计划的结构

一个好的计划包含：
1. **任务列表**：按执行顺序排列
2. **依赖关系**：哪些任务必须先完成
3. **完成标准**：每个任务做完是什么样子
4. **关注点**：每步需要调用哪些 skill

---

## 示例

### 示例 1：用户登录功能

```markdown
## 实现计划：用户邮箱登录

### 前置条件
- [x] 已安装 bcrypt、jsonwebtoken
- [ ] 数据库已连接

### 任务列表

**Step 1：设计数据库表**（依赖：无）
- 创建 users 表：id, email(唯一索引), password_hash, created_at
- 完成标准：migration 文件可以执行，表创建成功
- 调用：/db

**Step 2：实现密码工具函数**（依赖：Step 1）
- hashPassword(password) → hash
- comparePassword(password, hash) → boolean
- 完成标准：单元测试通过
- 调用：/tdd, /security

**Step 3：实现注册接口 POST /auth/register**（依赖：Step 1, 2）
- 验证邮箱格式、密码长度
- 检查邮箱是否已存在
- 存储哈希密码
- 完成标准：集成测试通过（正常注册 + 重复邮箱 = 409）

**Step 4：实现登录接口 POST /auth/login**（依赖：Step 2, 3）
- 查找用户
- 验证密码
- 返回 JWT（有效期 7 天）
- 完成标准：集成测试通过（正确密码 = 200+token，错误密码 = 401）

**Step 5：实现 JWT 验证中间件**（依赖：Step 4）
- 解析 Authorization: Bearer <token>
- 验证 token 有效性
- 注入 req.user
- 完成标准：受保护路由测试通过
```

### 示例 2：性能优化（并行任务）

```markdown
## 优化计划：首页加载速度

### 诊断阶段（先做，不可跳过）
**Step 1：跑 Lighthouse 基准测试**
- 记录当前 FCP、LCP、TTI 分数
- 完成标准：有数据基准可对比

### 优化阶段（Step 1 完成后，以下可并行）
**Step 2A：图片懒加载**
**Step 2B：拆分 JS Bundle（Code Splitting）**
**Step 2C：添加 HTTP 缓存头**

### 验证阶段（所有优化完成后）
**Step 3：再次跑 Lighthouse**
- 对比 Step 1 的数据
- 完成标准：LCP < 2.5s，TTI < 3.8s
```

---

## 好计划 vs 坏计划

| 坏计划 | 好计划 |
|--------|--------|
| "实现登录功能" | "Step 1: 建表，Step 2: 实现密码哈希，Step 3: 实现接口..." |
| 没有依赖关系 | 明确标注"依赖 Step X" |
| 没有完成标准 | 每步都有"完成标准：..." |
| 一个步骤太大 | 每步最多 2~4 小时内可完成 |
