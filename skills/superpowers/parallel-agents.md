# dispatching-parallel-agents

> 当有 2 个以上相互独立的任务时，同时派发给多个 AI agent 并行执行。

**类型**：流程类（Superpowers）
**触发时机**：有 2+ 个独立任务，且它们之间没有数据依赖

---

## 核心前提：任务必须真正独立

**独立的定义**：
- ✓ 修改的文件不重叠
- ✓ 不共享运行时状态
- ✓ A 完成与否，不影响 B 的执行

| 任务组合 | 能并行? |
|---------|--------|
| 重构 auth 模块 + 优化首页性能 | ✓ |
| 写单元测试 + 写 API 文档 | ✓ |
| 修改用户注册 + 修改用户登录 | ⚠️（共享 users 表，谨慎） |
| 修改数据库 schema + 修改使用该 schema 的接口 | ✗（有依赖） |

---

## 使用示例

### 示例 1：前后端并行开发

**任务**：同时实现登录功能的前后端

```
提前约定 API 接口契约（两者的连接点）：
POST /auth/login
  Request: { email: string, password: string }
  Response: { token: string, user: { id, email } }

Agent 1（后端）：
  - 实现 POST /auth/login 接口
  - 编写后端测试
  - 调用 security、db skill

Agent 2（前端）：
  - 实现登录表单组件
  - 实现 token 存储和请求拦截器
  - 用 mock 数据测试（不等后端完成）

两个 agent 同时工作，最后联调。
```

---

### 示例 2：多个独立模块的单元测试

**任务**：为用户模块、订单模块、通知模块各写测试

```
Agent 1 → tests/user.spec.ts
Agent 2 → tests/order.spec.ts
Agent 3 → tests/notification.spec.ts

三个测试文件完全独立，可以同时写。
```

---

### 示例 3：代码库多处重构

**任务**：把项目中所有的 `var` 改成 `const`/`let`，同时把所有回调改成 async/await

```
这两个任务可能有文件重叠，需要谨慎！

更好的做法：
  先完成一个（var → const/let），
  commit，
  再开始另一个（callback → async/await）。
```

---

## 与 subagent-driven-development 的区别

| | dispatching-parallel-agents | subagent-driven-development |
|---|---|---|
| 执行位置 | 多个外部 agent 进程 | 当前 session 内 |
| 真正并行 | ✓ 同时执行 | ✗ 顺序执行 |
| 适合场景 | 任务量大，可以真正并行 | 任务较小，在一个对话内完成 |
| 上下文 | 各 agent 独立上下文 | 共享同一个上下文 |

---

## 注意事项

1. **合并时注意冲突**：并行任务完成后，合并时可能有冲突，需要手动解决
2. **测试要分开跑**：各任务完成后先在自己的分支跑测试，合并后再全量测试
3. **沟通 API 契约**：前后端并行时，必须提前约定接口格式，否则联调时会有问题
