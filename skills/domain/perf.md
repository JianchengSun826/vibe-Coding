# perf（Domain Skill）

> 性能诊断、优化策略、前后端性能问题处理。

**类型**：领域类 — 按需查阅
**触发时机**：接口响应慢、页面加载慢、内存占用高、遇到性能问题

---

## 核心原则：先测量，再优化

```
❌ 错误：凭感觉猜哪里慢，然后去"优化"
✓ 正确：先 profiling 找到真正的瓶颈，再优化瓶颈

过早优化是万恶之源。
```

---

## 后端性能诊断

### 找慢 API

```bash
# 记录每个请求的响应时间（使用 morgan 或自定义中间件）
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    if (duration > 1000) {  // 超过 1 秒就记录
      console.warn(`Slow request: ${req.method} ${req.path} - ${duration}ms`);
    }
  });
  next();
});
```

### 找慢 SQL

```sql
-- PostgreSQL：查看慢查询日志
-- postgresql.conf 开启：log_min_duration_statement = 1000  (超过 1 秒记录)

-- 分析某个查询
EXPLAIN ANALYZE
SELECT orders.*, users.name
FROM orders
JOIN users ON orders.user_id = users.id
WHERE orders.status = 'pending';

-- 关注：
-- "Seq Scan" → 全表扫描，可能需要索引
-- "rows=10000" 但 "actual rows=1" → 统计信息过期，需要 ANALYZE
```

### N+1 问题（最常见）

见 `/db` skill。

---

## 前端性能诊断

### Lighthouse

```bash
# 命令行
npx lighthouse http://localhost:3000 --view

# 或在 Chrome DevTools → Lighthouse 标签
```

关键指标：
| 指标 | 好 | 差 | 含义 |
|------|----|----|------|
| FCP | < 1.8s | > 3s | 第一内容绘制 |
| LCP | < 2.5s | > 4s | 最大内容绘制 |
| TTI | < 3.8s | > 7.3s | 可交互时间 |
| CLS | < 0.1 | > 0.25 | 布局稳定性 |

### Chrome DevTools Performance

```
1. 打开 DevTools → Performance 标签
2. 点击 Record，操作页面，点击 Stop
3. 查看：
   - 黄色块（JS 执行）→ 长时间黄色 = JS 阻塞
   - 紫色块（Layout/Reflow）→ 频繁 Layout = 性能差
   - 网络瀑布图 → 哪些资源加载慢
```

---

## 常见优化策略

### 1. 数据库查询优化

```sql
-- 添加索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 只查需要的字段（避免 SELECT *）
SELECT id, status, total FROM orders WHERE user_id = $1;

-- 用 LIMIT 限制结果数量
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;
```

### 2. 缓存

```typescript
// 内存缓存（适合单机，数据不敏感）
const cache = new Map<string, { data: any; expiresAt: number }>();

async function getUserWithCache(userId: number) {
  const cacheKey = `user:${userId}`;
  const cached = cache.get(cacheKey);

  if (cached && cached.expiresAt > Date.now()) {
    return cached.data;  // 命中缓存
  }

  const user = await db.user.findById(userId);  // 查数据库
  cache.set(cacheKey, {
    data: user,
    expiresAt: Date.now() + 5 * 60 * 1000,  // 5 分钟缓存
  });
  return user;
}
```

### 3. 前端 Bundle 优化

```javascript
// ❌ 一次性加载所有代码
import HeavyChart from 'heavy-chart-library';  // 300KB

// ✓ 懒加载（只有用到时才下载）
const HeavyChart = React.lazy(() => import('heavy-chart-library'));

// ✓ 路由级别代码分割
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));
```

### 4. 图片优化

```html
<!-- ❌ 不做任何优化 -->
<img src="hero.png" />

<!-- ✓ 懒加载 + 现代格式 + 响应式 -->
<img
  src="hero.webp"
  loading="lazy"
  width="800" height="400"
  srcset="hero-400.webp 400w, hero-800.webp 800w"
  sizes="(max-width: 400px) 400px, 800px"
  alt="Hero image"
/>
```

### 5. HTTP 缓存

```typescript
// 静态资源（JS/CSS/图片）：长期缓存 + 文件名 hash
// webpack/vite 会自动给文件名加 hash：main.a1b2c3.js

// API 响应缓存头
app.get('/api/categories', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=300');  // 缓存 5 分钟
  res.json(categories);
});
```

---

## 优化前后要对比数据

```
优化前：
  API /api/orders 响应时间：平均 1.8s，P99 4.2s
  数据库查询：15 次（N+1 问题）

优化后（添加 JOIN + 索引）：
  API /api/orders 响应时间：平均 0.12s，P99 0.3s
  数据库查询：1 次

提升：15x 速度提升
```

没有数据对比，不叫"优化"，叫"修改"。
