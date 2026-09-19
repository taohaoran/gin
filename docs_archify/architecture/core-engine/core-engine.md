# 核心引擎（core-engine）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin`（Go 1.26 指令），commit `5c6a15f`。

## 1. 域职责

core-engine 是 gin 框架的"请求处理核心"：从 `net/http` 接收请求，复用池化 Context，把请求派发到路由树，驱动中间件+业务 handler 链，并通过拦截 `http.ResponseWriter` 捕获状态码与字节数。本域覆盖引擎生命周期、请求上下文对象、路由分组注册与响应写出器四部分。

核心代码路径：
- 引擎入口：`gin.go`（`Engine.ServeHTTP` → `handleHTTPRequest`）
- 请求上下文：`context.go`（`Context` 与全部方法族）
- 路由注册：`routergroup.go`（`RouterGroup` 注册 API）
- 响应写出：`response_writer.go` + `fs.go`

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| engine-lifecycle | [engine-lifecycle.md](engine-lifecycle/engine-lifecycle.md) | [架构图](engine-lifecycle/engine-lifecycle-architecture.html) | [时序图](engine-lifecycle/engine-lifecycle-sequence.html) | Engine 构建/启动/ServeHTTP 派发/可信代理，sync.Pool+sync.Once |
| context-object | [context-object.md](context-object/context-object.md) | [架构图](context-object/context-object-architecture.html) | — | Context 结构、链流控、取值/绑定/渲染、context.Context 适配 |
| router-group | [router-group.md](router-group/router-group.md) | [架构图](router-group/router-group-architecture.html) | — | RouterGroup 注册 API、分组嵌套、中间件合并、静态文件挂载 |
| response-writer | [response-writer.md](response-writer/response-writer.md) | [架构图](response-writer/response-writer-architecture.html) | — | 拦截 http.ResponseWriter 捕获 Status/Size、延迟落头、关目录列表 |

## 3. 域级机制细节

- **Engine 与 Context/ContextPool 的关系**：`Engine.pool sync.Pool`（`gin.go:183`）在 `New()` 时配置 `pool.New = allocateContext`（`gin.go:229`），预分配 `Params` 与 `skippedNodes` 容量。`ServeHTTP`（`gin.go:662`）每请求 `pool.Get()` 取 Context → `c.writermem.reset(w)` → `c.reset()` → 派发 → `pool.Put(c)`。`Context.reset()`（`context.go:103`）把切片长度归零但保留底层容量，实现零分配复用。
- **注册期写、运行期读**：所有路由注册（`addRoute`）在 `Run` 前单 goroutine 完成；`sync.Once`（`gin.go:97`）保证首个请求时才把 `\:` 转义还原。运行期 `handleHTTPRequest` 只读遍历路由树。
- **中间件"烤进"路由链**：`RouterGroup.combineHandlers`（`routergroup.go:251`）在注册期把全局/组中间件与路由 handler 合并成一条 `HandlersChain`，运行期 `Context.Next()`（`context.go:198`）用 `index` 游标顺序驱动，零查找——这是 gin 高性能的关键设计。
- **响应写出拦截**：`Context.Writer` 类型是 `ResponseWriter` 接口（`response_writer.go:23`），底层 `responseWriter` 延迟落头（先记录 status，`WriteHeaderNow` 才真正下发），使中间件能在 `Next()` 之后读到真实状态码。

## 4. 与相邻域的边界

- 下游依赖 `router-tree` 域：`Engine.handleHTTPRequest` 调 `radix-tree` 的 `getValue`/`findCaseInsensitivePath`，`router-group` 调 `path-utils` 的 `joinPaths`。
- 本域不实现路由树内部、绑定器（`binding/`）、渲染器（`render/`）——这些是被调用的外部子系统/包。
