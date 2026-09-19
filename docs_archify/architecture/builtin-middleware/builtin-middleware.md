# 内置中间件（builtin-middleware）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` commit `5c6a15f`。

## 1. 域职责

本域覆盖 Gin 框架内置、可通过 `RouterGroup.Use(...)` 装配到 handler 链上的四个中间件：访问日志、panic 恢复、HTTP Basic 认证、路由调试打印。它们在 `gin.Default()` 中默认挂载 `Logger()` + `Recovery()`；认证与调试打印由用户按需启用。

核心执行机制：中间件是 `HandlerFunc`（`func(c *Context)`），挂到 `RouterGroup.Handlers` 后，`Engine.handleHTTPRequest` 把它与路由 handler 合并成 `HandlersChain`；请求进入后 `Context.Next()`（`context.go:198`）自增 `index` 顺序执行链上每个函数。中间件通过在 `c.Next()` 前后插桩（计时、recover defer、认证判定）实现横切关注点，用 `c.Abort()/AbortWithStatus()`（`context.go:217`、`context.go:223`）提前终止链。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| logger-middleware | [logger-middleware.md](logger-middleware/logger-middleware.md) | [架构图](logger-middleware/logger-middleware-architecture.html) | — | 请求级访问日志：计时、状态码、ClientIP、可跳过、彩色 |
| recovery-middleware | [recovery-middleware.md](recovery-middleware/recovery-middleware.md) | [架构图](recovery-middleware/recovery-middleware-architecture.html) | [时序图](recovery-middleware/recovery-middleware-sequence.html) | defer recover 捕获下游 panic，打脱敏栈并返回 500 |
| basic-auth | [basic-auth.md](basic-auth/basic-auth.md) | [架构图](basic-auth/basic-auth-architecture.html) | — | HTTP Basic/Proxy 认证，常量时间比对，回写用户名 |
| debug-print | [debug-print.md](debug-print/debug-print.md) | [架构图](debug-print/debug-print-architecture.html) | — | 仅 Debug 模式打印路由注册与模板加载 |

## 3. 域级机制细节

- **中间件装配顺序**：`Engine.Default()`（`gin.go:236`）调用 `New()` 后 `Use(Logger(), Recovery())`（`gin.go:239`）。顺序保证 Logger 先计时、Recovery 的 defer 先注册，panic 发生时 Recovery 兜底。
- **共享 Context**：四个中间件都操作同一个 `*Context`——Logger 读 `c.Errors`/`c.Writer`，Recovery 写 `c.Errors` 并 Abort，BasicAuth 写 `c.Keys[AuthUserKey]`，debug-print 不参与请求期。
- **可见性分离**：私有错误（`ErrorTypePrivate`）进访问日志，公开错误（`ErrorTypePublic`）可经 `ErrorLoggerT` 返回客户端（见 error-model 叶子）。
- **输出统一**：Logger 默认写 `DefaultWriter`，Recovery 写 `DefaultErrorWriter`（见 runtime-mode 叶子）。

## 4. 域级图

本域不单独绘制域级架构图：四个中间件共享同一套 Context/Next/Abort 执行链，其关系已在系统级架构图与各叶子图中表达；避免与叶子图重复。
