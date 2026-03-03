# db（Domain Skill）

> 数据库表设计、查询优化、迁移管理、N+1 问题。

**类型**：领域类 — 按需查阅
**触发时机**：设计数据库表、写 SQL 查询、遇到慢查询、处理数据库迁移

---

## 表设计原则

### 命名规范

```sql
-- 表名：复数、snake_case
users, orders, product_categories

-- 列名：snake_case
user_id, created_at, password_hash

-- 主键：id（自增或 UUID）
-- 外键：<表名单数>_id，例如 user_id, order_id
```

### 基础字段（每张表都该有的）

```sql
CREATE TABLE users (
  id          SERIAL PRIMARY KEY,       -- 或 UUID
  created_at  TIMESTAMP DEFAULT NOW(),
  updated_at  TIMESTAMP DEFAULT NOW()
);
```

### 示例：用户表设计

```sql
CREATE TABLE users (
  id            SERIAL PRIMARY KEY,
  email         VARCHAR(255) UNIQUE NOT NULL,  -- 唯一索引
  password_hash VARCHAR(255) NOT NULL,          -- 不存明文！
  name          VARCHAR(100),
  role          VARCHAR(20) DEFAULT 'user',     -- 'user', 'admin'
  created_at    TIMESTAMP DEFAULT NOW(),
  updated_at    TIMESTAMP DEFAULT NOW()
);

-- email 查询很频繁，需要索引
CREATE INDEX idx_users_email ON users(email);
```

---

## 索引

### 什么时候加索引

```sql
-- ✓ 应该加索引的情况：
-- 1. WHERE 子句中频繁用到的字段
-- 2. JOIN 的关联字段
-- 3. ORDER BY 的字段（尤其是分页）
-- 4. UNIQUE 约束（会自动创建唯一索引）

-- 示例
CREATE INDEX idx_orders_user_id ON orders(user_id);       -- 查用户订单
CREATE INDEX idx_orders_created_at ON orders(created_at); -- 按时间排序
CREATE INDEX idx_orders_status ON orders(status)
  WHERE status = 'pending';                                -- 部分索引（只索引待处理订单）
```

### 诊断是否需要索引

```sql
-- 用 EXPLAIN ANALYZE 查看查询计划
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123;

-- 关注：
-- "Seq Scan"（全表扫描）→ 可能需要索引
-- "Index Scan" → 已经用到索引了
-- "cost=xxx..xxx" → 越小越好
-- "actual time=xxx..xxx" → 实际执行时间
```

---

## N+1 查询问题

这是最常见的性能问题：在循环里发 N 次数据库请求。

```typescript
// ❌ N+1 问题：查 1 次订单列表 + N 次用户信息
const orders = await db.query('SELECT * FROM orders');  // 1 次查询

for (const order of orders) {
  // N 次查询！100 个订单 = 101 次数据库请求
  order.user = await db.query('SELECT * FROM users WHERE id = $1', [order.userId]);
}

// ✓ 解决方案 1：JOIN 一次性查询
const orders = await db.query(`
  SELECT orders.*, users.name, users.email
  FROM orders
  LEFT JOIN users ON orders.user_id = users.id
`);

// ✓ 解决方案 2：先批量查，再关联（适合 ORM）
const orders = await Order.findAll();
const userIds = orders.map(o => o.userId);
const users = await User.findAll({ where: { id: userIds } });  // 1 次查询
const userMap = Object.fromEntries(users.map(u => [u.id, u]));
orders.forEach(o => o.user = userMap[o.userId]);
```

---

## 分页

```sql
-- ❌ OFFSET 分页（数据量大时越来越慢）
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 1000000;
-- 数据库需要扫描并跳过 100 万行

-- ✓ 游标分页（Keyset Pagination，稳定高效）
SELECT * FROM orders
WHERE id > :last_id     -- 上次返回的最后一个 id
ORDER BY id
LIMIT 20;
```

---

## 数据库迁移

```bash
# 使用 Prisma 示例
npx prisma migrate dev --name add_users_table
npx prisma migrate deploy    # 生产环境执行迁移
npx prisma db pull           # 从数据库同步 schema

# 使用 Knex 示例
npx knex migrate:make add_users_table
npx knex migrate:latest
npx knex migrate:rollback    # 回滚
```

迁移文件示例（不可逆操作要谨慎）：
```javascript
// migrations/20240101_add_users_table.js
exports.up = (knex) => {
  return knex.schema.createTable('users', (table) => {
    table.increments('id');
    table.string('email', 255).unique().notNullable();
    table.string('password_hash', 255).notNullable();
    table.timestamps(true, true);
  });
};

exports.down = (knex) => {
  return knex.schema.dropTable('users');
};
```

---

## 常见查询模式

```sql
-- 查找（精确匹配）
SELECT * FROM users WHERE email = 'user@example.com';

-- 模糊搜索
SELECT * FROM products WHERE name ILIKE '%search_term%';
-- 注意：LIKE 查询不能用普通索引，需要 Full Text Search 或 pg_trgm

-- 聚合
SELECT status, COUNT(*) as count
FROM orders
GROUP BY status;

-- 最近 7 天的订单
SELECT * FROM orders
WHERE created_at >= NOW() - INTERVAL '7 days'
ORDER BY created_at DESC;

-- 事务（要么都成功，要么都回滚）
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- 如果中间出错：ROLLBACK;
```
