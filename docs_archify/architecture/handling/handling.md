# 请求处理（handling）域总览

> 本域是 gin 框架的核心运行时：一个 HTTP 请求从进入 `Engine.ServeHTTP` 到写出响应的完整链路。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`。

## 1. 域职责

本域覆盖 gin 作为 HTTP Web 框架的**请求处理主循环**，包含三个紧耦合叶子：

- **engine**（`gin.go`/`mode.go`/`debug.go`/`version.go`）：`Engine` 实例、Run 家族监听入口、`ServeHTTP`/`handleHTTPRequest` 请求分发、可信代理校验、HTML 模板加载、模式与调试。
- **context**（`context.go`）：每请求 `Context`——中间件链 `Next/Abort` 协议、参数/Query/Form 取值、绑定（Bind/ShouldBind）、渲染快捷（JSON/HTML/Stream/File）、客户端 IP、context.Context 适配。
- **response-writer**（`response_writer.go`）：`ResponseWriter` 接口与 `responseWriter` 对标准库 writer 的包装——status/size 跟踪、延迟写头、Hijack/Flush/CloseNotify/Pusher 透传。

域内共享机制：
- **`sync.Pool` 复用 Context**（`gin.go:183/667`）：每请求 Get/Put，避免 Context 及其 params 切片反复分配。
- **洋葱模型中间件链**：`c.Next()` 用 `index int8` 游标推进，`Abort()` 把 index 跳到 `abortIndex` 阻断后续（`context.go:198/217`）。
- **延迟写头**：`WriteHeader` 只记 status，首次 `Write`/`WriteHeaderNow` 才真正下发（`response_writer.go:67/77`）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序 / 其他图 | 职责一句话 |
|---|---|---|---|---|
| engine | [engine.md](engine/engine.md) | [架构图](engine/engine-architecture.html) | [请求分发时序](engine/engine-sequence.html) | Engine 实例、Run 家族、请求分发、可信代理 |
| context | [context.md](context/context.md) | [结构图](context/context-architecture.html) | [中间件链时序](context/context-sequence.html) · [绑定渲染数据流](context/context-dataflow.html) | 每请求上下文：链式执行、取值、绑定、渲染 |
| response-writer | [response-writer.md](response-writer/response-writer.md) | [架构图](response-writer/response-writer-architecture.html) | [写入状态机](response-writer/response-writer-lifecycle.html) | 包装标准库 writer，跟踪 status/size，透传 Hijack/Flush |

## 3. 域级机制细节

### 3.1 请求生命周期总览
`net/http.Server` 每请求起 goroutine → `Engine.ServeHTTP`（`gin.go:662`）→ `pool.Get` 取 Context → `handleHTTPRequest` 查路由树 → 命中则 `c.Next()` 执行中间件链 → `WriteHeaderNow` 兜底落头 → `pool.Put` 回收。未命中走尾随斜杠/固定路径重定向 → 405 → 404。

### 3.2 Context 对象池与复用
`Engine.pool sync.Pool`（`gin.go:183`）的 `New` 回调是 `allocateContext(maxParams)`（`gin.go:252`），按已注册路由的最大参数数预分配 `Params` 与 `skippedNodes` 容量。`c.reset()`（`context.go:103`）在 Get 后清零所有字段。跨 goroutine 必须用 `c.Copy()`（`context.go:122`）深拷贝。

### 3.3 可信代理与客户端 IP
默认信任全部代理（`0.0.0.0/0`+`::/0`），启动时打印告警。生产应 `SetTrustedProxies([]string{"10.0.0.0/8"})`。`Context.ClientIP`（`context.go:998`）优先 TrustedPlatform，再判断远端是否 trusted proxy，是才反解 X-Forwarded-For。

### 3.4 绑定与渲染的依赖倒置
Context 面向 `binding.Binding`（`binding/binding.go`）与 `render.Render`（`render/render.go`）两个接口编程，具体 JSON/XML/Form/Uri 实现与 JSON/HTML/Data 渲染器都在子包，本域只做分发。

## 4. 域级图

![handling 域架构图](handling-architecture.html)

![请求生命周期时序](handling-sequence.html)

![请求处理数据流](handling-dataflow.html)
