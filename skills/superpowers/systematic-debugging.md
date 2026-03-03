# systematic-debugging

> 通过证据系统性地找到根因，而不是靠猜测随机尝试修复。

**类型**：流程类（Superpowers）— 不可跳过
**触发时机**：遇到任何 bug、测试失败、或意料之外的行为

---

## 核心流程

```
1. 复现问题      ← 不能稳定复现 = 不能确认已修复
2. 收集证据      ← 日志、错误信息、堆栈追踪
3. 提出假设      ← 基于证据，不是直觉
4. 验证假设      ← 修改一个变量，观察结果
5. 确认根因      ← 找到根本原因，而不是症状
6. 修复 + 测试   ← 写复现测试（/tdd），再修复
```

---

## 示例

### 示例 1：接口偶尔返回 500

```
问题描述："POST /auth/login 偶尔返回 500"

Step 1 复现：
  - 用正常账号登录，大部分成功
  - 高并发下（压测 100 rps）每隔几十个请求失败一次
  - 复现条件：高并发 + 数据库负载高

Step 2 收集证据：
  - 服务器日志：TypeError: Cannot read property 'id' of undefined
  - 堆栈：at AuthService.login (auth.service.ts:47)
  - Line 47：const token = generateToken(user.id);
  - 数据库日志：某些时刻出现 "Query timeout after 5000ms"

Step 3 提出假设：
  - 假设A：数据库查询超时时，findUser 返回 undefined，
           调用方没有 null 检查就直接访问 .id → 崩溃
  - 假设B（排除）：内存泄漏导致崩溃 — 不符合，重启后相同情况仍然触发

Step 4 验证假设A：
  - 查看 findUser 实现：超时时 catch 块里 return undefined（确认）
  - 查看调用方：直接 user.id，无 null 检查（确认）
  - 本地模拟：mock findUser 返回 undefined → 复现 TypeError（确认）

Step 5 根因：
  数据库超时 → findUser 返回 undefined → 无 null 检查 → TypeError → 500

Step 6 修复：
  if (!user) throw new UnauthorizedException('Invalid credentials');
  写复现测试 → 修复 → 验证测试通过
```

---

### 示例 2：测试偶尔失败（Flaky Test）

```
问题："UserService 测试有时通过有时失败，很随机"

Step 1 复现：
  - 单独跑：总是通过
  - 跑完整测试套件：偶尔失败
  - 连续跑 10 次完整套件：失败 3 次

Step 2 收集证据：
  - 失败的错误：expected 5 users, got 8
  - 失败的测试：it('should return all users', ...)
  - 观察：只有在其他测试之后才失败

Step 3 提出假设：
  - 假设：测试之间有状态污染
           某个先跑的测试往数据库插了用户，但没清理

Step 4 验证：
  - 在失败测试前加 log，打印当前数据库中的 users
  - 发现有 3 条不属于本测试的记录（来自 AuthService 测试插入的测试用户）

根因：AuthService 测试插入了用户，但 teardown 没有清理数据库
修复：在 afterEach 或 afterAll 中清理测试数据库
```

---

### 示例 3：性能突然变慢

```
问题："首页从昨天开始加载变慢了，从 0.5s 变成 3s"

Step 2 收集证据：
  - 对比昨天和今天的 git log，找昨天的提交
  - 发现：昨天合并了一个 PR，修改了 /api/home 接口
  - 用 curl 测试接口响应时间：2.8s（API 慢）
  - 查看数据库慢查询日志：一个查询耗时 2.5s

Step 3 假设：
  - 假设：PR 里的改动引入了慢查询

Step 4 验证：
  - EXPLAIN ANALYZE 分析该查询
  - 发现：products 表有 100 万条数据，新增的 WHERE 子句用了没有索引的字段

根因：新 WHERE 条件没有对应索引 → 全表扫描 → 2.5s
修复：为该字段添加索引
```

---

## 禁止的做法

```
❌ "可能是 X，我改一下试试" — 没有证据的猜测
❌ 同时修改多个地方 — 无法判断哪个修改有效
❌ "重启一下服务器" — 治标不治本，问题还会回来
❌ "删除缓存试试" — 没有证据显示是缓存问题
❌ 不写复现测试就修复 — 无法验证真的修好了
```

## 好用的调试工具

| 场景 | 工具 |
|------|------|
| 后端日志 | console.log、结构化日志（pino、winston） |
| 数据库慢查询 | EXPLAIN ANALYZE、慢查询日志 |
| 前端性能 | Chrome DevTools → Performance tab |
| 测试调试 | jest --verbose、单独跑某个测试 |
| 网络请求 | curl -v、Postman、Chrome Network tab |
