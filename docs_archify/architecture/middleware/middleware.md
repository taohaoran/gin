# middleware（中间件）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 域职责

gin 的中间件域实现"处理器链"这一 Web 框架核心抽象：把横切关注点（日志、崩溃恢复、鉴权、静态文件服务）组织成可复用的 `HandlerFunc` 序列，在请求期由 `Context.Next()` 用 `int8` 游标顺序驱动。本域不做业务路由匹配（基数树属路由域），也不做响应渲染（属 render 域）；它定义**中间件如何被注册、组合、链式执行与截断**，并内置三类常用中间件（日志、崩溃恢复、BasicAuth）与静态文件服务。

gin 没有独立的 middleware.go：链协议在 `gin.go`/`routergroup.go`/`context.go`，三个内置中间件各自独立成文件（`logger.go`/`recovery.go`/`auth.go`），静态文件在 `fs.go` + `routergroup.go`。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 其他图 | 职责一句话 |
|------|------|--------|--------|--------|-----------|
| 域总览 | 本文 | [架构图](middleware-architecture.html) | [时序图](middleware-sequence.html) | [数据流图](middleware-dataflow.html) | 中间件链体系总览 |
| middleware-core | [middleware-core.md](middleware-core/middleware-core.md) | [架构图](middleware-core/middleware-core-architecture.html) | [时序图](middleware-core/middleware-core-sequence.html) | [状态机](middleware-core/middleware-core-lifecycle.html) | Use/Group 注册、Next/Abort 链协议、静态文件服务 |
| logger | [logger.md](logger/logger.md) | [架构图](logger/logger-architecture.html) | [时序图](logger/logger-sequence.html) | — | 访问日志中间件与可插拔格式化 |
| recovery-auth | [recovery-auth.md](recovery-auth/recovery-auth.md) | [架构图](recovery-auth/recovery-auth-architecture.html) | [时序图](recovery-auth/recovery-auth-sequence.html) | [状态机](recovery-auth/recovery-auth-lifecycle.html) | panic 崩溃恢复 + BasicAuth 鉴权 |

## 3. 域级机制细节

- **统一的链游标协议**：所有中间件共享同一套 `Context.index int8` 游标。`c.Next()` 递增游标跑循环；`c.Abort()` 把游标打到 `abortIndex`（`math.MaxInt8>>1`）截断后续。中间件靠"在自身内部先 `c.Next()`"把下游包起来——这是 Logger 能计时、Recovery 能 defer recover 的根本机制。
- **注册期烘焙**：`combineHandlers`（`routergroup.go:251`）在路由注册时就把"全局中间件 + 组中间件 + 路由处理器"拼成一条 `HandlersChain` 插入基数树；请求期只取出执行，不再拼接。
- **`Default()` 的默认组合**：`gin.Default()`（`gin.go:236`）= `New()` + `Use(Logger(), Recovery())`，即日志与崩溃恢复默认挂在链首。
- **中间件间通过 `c.Keys` 与 `c.Errors` 解耦**：BasicAuth 把用户名写进 `c.Keys[AuthUserKey]`；Logger/Recovery 读 `c.Errors`（support 域 errors 叶子）——三者无直接调用，靠 Context 共享数据。

## 4. 域级图

![middleware 域架构图](middleware-architecture.html)

![跨中间件链时序](middleware-sequence.html)

![中间件链请求数据流](middleware-dataflow.html)

**省略说明**：本域未额外生成 lifecycle 域图——单实体状态机语义已在 middleware-core（游标生命周期）与 recovery-auth（panic 处置）两个叶子级 lifecycle 图表达，域层再画会重复，按资源节省原则省略。
