# 引擎生命周期（engine-lifecycle）

> 本文是 `core-engine` 域下的叶子子系统文档。域级总览见 `../core-engine.md`。
> 本文只展开 **Engine 实例的构建、HTTP 入口生命周期、请求派发主路径与可信代理校验**，不重复展开路由树匹配内部细节（见 `../../router-tree/radix-tree/radix-tree.md`）、Context 字段语义（见 `../context-object/context-object.md`）与路由分组注册（见 `../router-group/router-group.md`）。
>
> 源码基准：`github.com/gin-gonic/gin`（Go 1.26 指令，实际要求 Go 1.25+），commit `5c6a15f`。主源码文件：`gin.go`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Engine 实例构建 | `New()` 空引擎、`Default()` 挂 Logger+Recovery、`With(opts...)` 选项式改配置 | `gin.go:202`（New）、`gin.go:236`（Default）、`gin.go:348`（With） |
| RouterGroup 内嵌 | Engine 通过匿名内嵌 `RouterGroup` 同时实现 `IRouter`，对外暴露分组/注册能力 | `gin.go:93` |
| 启动方法族 | `Run/RunTLS/RunUnix/RunFd/RunQUIC/RunListener`，均包一层 `http.Server` 后阻塞服务 | `gin.go:540`（Run）、`gin.go:561`（RunTLS）、`gin.go:581`（RunUnix）、`gin.go:607`（RunFd）、`gin.go:630`（RunQUIC）、`gin.go:645`（RunListener） |
| H2C 升级 | `Handler()` 在 `UseH2C=true` 时用 `h2c.NewHandler` 包装引擎，支持明文 HTTP/2 | `gin.go:243` |
| HTTP 入口 | `ServeHTTP(w, req)` 实现 `http.Handler`：取池化 Context → reset → 派发 → 归还 | `gin.go:662` |
| 请求派发 | `handleHTTPRequest(c)`：选 method 树 → 路由匹配 → 命中执行链 / TSR 重定向 / 固定路径重定向 / 405 / 404 | `gin.go:690` |
| 404/405 错误响应 | `serveError` 写默认 plain body；405 按 RFC 7231 拼 `Allow` 头 | `gin.go:764`、`gin.go:738` |
| 重定向 | `redirectTrailingSlash`（尾斜杠）、`redirectFixedPath`（大小写/多余斜杠修复）、`redirectRequest`（GET→301，其余→307） | `gin.go:781`、`gin.go:808`、`gin.go:820` |
| Context 池化 | `pool sync.Pool` + `allocateContext` 预分配 Params/skippedNodes 容量 | `gin.go:183`、`gin.go:229`、`gin.go:252` |
| 路由树一次性收尾 | `routeTreesUpdated sync.Once` 首次请求时把 `\:` 转义还原为 `:` | `gin.go:97`、`gin.go:663`、`gin.go:517` |
| 可信代理 / 客户端 IP | `SetTrustedProxies` → `prepareTrustedCIDRs`/`parseTrustedProxies`，`validateHeader` 逆序解析 X-Forwarded-For | `gin.go:451`、`gin.go:414`、`gin.go:482` |
| 路由注册入口 | `addRoute`：校验路径/方法/handler、按 method 找根节点、调用 `root.addRoute`，统计 maxParams/maxSections | `gin.go:364` |
| 404/405 handler 重建 | `Use/NoRoute/NoMethod` 触发 `rebuild404Handlers/rebuild405Handlers` 合并全局中间件 | `gin.go:340`、`gin.go:326`、`gin.go:332`、`gin.go:356` |
| 路由清单导出 | `Routes()` + `iterate` 遍历各 method 树，反射取 handler 名 | `gin.go:390`、`gin.go:397` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `Engine` struct | `gin.go:92` | 框架实例：内嵌 RouterGroup，持有 trees/pool/代理与重定向开关、404/405 链 |
| `HandlerFunc` | `gin.go:51` | `func(*Context)`，中间件与业务 handler 的统一签名 |
| `HandlersChain` | `gin.go:57` | `[]HandlerFunc`，`Last()` 取主 handler |
| `OptionFunc` | `gin.go:54` | `func(*Engine)`，`With()` 的选项式配置回调 |
| `RouteInfo` / `RoutesInfo` | `gin.go:68` / `gin.go:76` | `Routes()` 导出的路由描述（method/path/handler 名） |
| `methodTree` / `methodTrees` | 见 `tree.go:45` / `tree.go:50` | 每个 HTTP method 一棵树，`get(method)` 线性查找根节点 |
| `defaultTrustedCIDRs` | `gin.go:39` | 默认信任 `0.0.0.0/0` 与 `::/0`（即默认信任全部代理，`isUnsafeTrustedProxies` 据此告警） |
| `Handler() http.Handler` | `gin.go:243` | 满足 `http.Handler` 接口（Engine 本身也是）；声明 `var _ IRouter = (*Engine)(nil)` 在 `gin.go:191` |

## 3. 关键调用链

**链路 A：请求从 net/http 进入并派发（主路径）**

1. net/http 每连接 goroutine 调 `engine.ServeHTTP(w, req)`（`gin.go:662`）。
2. 首次请求经 `routeTreesUpdated.Do(updateRouteTrees)`（`gin.go:663`）把路由树里 `\:` 转义还原为 `:`，之后不再执行（`sync.Once`）。
3. `c := engine.pool.Get().(*Context)` 取复用 Context，`c.writermem.reset(w)`、`c.Request = req`、`c.reset()`（`gin.go:667-670`）。
4. `engine.handleHTTPRequest(c)`（`gin.go:672`）：按 `UseEscapedPath/UseRawPath/RemoveExtraSlash` 计算 `rPath`（`gin.go:695-705`）。
5. 线性遍历 `engine.trees` 找到同 method 根节点，调 `root.getValue(rPath, c.params, c.skippedNodes, unescape)`（`gin.go:715`）。
6. 命中（`value.handlers != nil`）：`c.handlers = value.handlers`、`c.Next()` 顺序执行整条 handler 链，结束后 `c.writermem.WriteHeaderNow()`（`gin.go:719-724`）。
7. 未命中：按 `value.tsr && RedirectTrailingSlash` → `redirectTrailingSlash`；否则 `RedirectFixedPath` → `redirectFixedPath`；再不行进入 405/404 分支（`gin.go:726-759`）。
8. 回到 `ServeHTTP`，`engine.pool.Put(c)` 归还 Context（`gin.go:674`）。

**链路 B：重新进入一个被改写的 Context（内部重入）**

- `HandleContext(c)` 先保存 `oldIndexValue/oldHandlers`，`c.reset()` 后 `handleHTTPRequest(c)`，再恢复旧 index/handlers（`gin.go:680-688`），供开发者改 `c.Request.URL.Path` 后内部转发。

**链路 C：可信代理校验客户端 IP**

1. `SetTrustedProxies([]string)` 赋值并调 `parseTrustedProxies()` → `prepareTrustedCIDRs()`，把纯 IP 补 `/32` 或 `/128`，`net.ParseCIDR` 解析为 `[]*net.IPNet`（`gin.go:451`、`gin.go:462`、`gin.go:414`）。
2. 运行期 `Context.ClientIP()`（见 `context.go:998`）先判 `TrustedPlatform`/unix socket 信任，再用 `isTrustedProxy(remoteIP)`（`gin.go:469`）判断远端是否可信代理。
3. 若可信且 `ForwardedByClientIP`，遍历 `RemoteIPHeaders`（默认 `[X-Forwarded-For, X-Real-IP]`），`validateHeader` **从右往左**解析 XFF，遇到第一个不可信代理即停并返回该 IP（`gin.go:482-500`）。

## 4. 配置项

| 配置项 / flag | 默认 / 行为 | 位置 |
|------|------|------|
| `RedirectTrailingSlash` | 默认 `true`，尾斜杠自动重定向 | `gin.go:211` |
| `RedirectFixedPath` | 默认 `false`，大小写/多余斜杠修复重定向 | `gin.go:212` |
| `HandleMethodNotAllowed` | 默认 `false`，开启后未命中走 405 | `gin.go:213` |
| `ForwardedByClientIP` | 默认 `true`，从代理头解析客户端 IP | `gin.go:214` |
| `RemoteIPHeaders` | 默认 `[X-Forwarded-For, X-Real-IP]` | `gin.go:215` |
| `TrustedProxies/TrustedPlatform` | 默认信任全部 `0.0.0.0/0,::/0`；`TrustedPlatform` 为空 | `gin.go:225`、`gin.go:216` |
| `UseRawPath` / `UseEscapedPath` | 默认 `false`；后者覆盖前者，决定用 `RawPath/EscapedPath` 找参 | `gin.go:217`、`gin.go:144` |
| `UnescapePathValues` | 默认 `true`，路由参数是否 URL 反转义 | `gin.go:220` |
| `RemoveExtraSlash` | 默认 `false`，开启后 `rPath = cleanPath(rPath)` | `gin.go:219`、`gin.go:703` |
| `MaxMultipartMemory` | 默认 32MB（`32 << 20`），透传给 `ParseMultipartForm` | `gin.go:26`、`gin.go:221` |
| `UseH2C` | 默认 `false`，开启后 `Handler()` 包 h2c | `gin.go:170`、`gin.go:244` |
| `ContextWithFallback` | 默认 `false`，开启后 Context 自身实现 `context.Context` 时回退到 Request.Context() | `gin.go:173`、`context.go:1495` |
| `PORT` 环境变量 | `Run()` 未传地址时读 `PORT`，缺省回退 `:8080` | `utils.go:148` |

## 5. 错误与重试语义

- **路由派发无重试**：`handleHTTPRequest` 是确定性查表，未命中直接走重定向/405/404，不做重试。
- **405 拼装**：`HandleMethodNotAllowed` 开启时，对每个其它 method 树再做一次 `getValue`（传 `nil` params）判断该路径是否支持该方法，收集到 `Allow` 头（`gin.go:741-753`）。
- **serveError 兜底**：先 `c.Next()` 跑 404/405 自定义链；若仍未写 body 且状态码未变，写 `default404Body/default405Body` 并设 `text/plain`（`gin.go:764-778`）。
- **启动错误**：所有 `Run*` 用 `defer func(){ debugPrintError(err) }()` 打印监听错误后原样返回给调用方；不自动重试绑定端口。
- **可信代理不安全告警**：`isUnsafeTrustedProxies()` 命中 `0.0.0.0`/`::` 时在 `Run*` 里 `debugPrint` 警告（`gin.go:543-546`）。
- **SetTrustedProxies 错误**：CIDR/IP 解析失败通过 `net.ParseError`/`net.ParseCIDR` 错误返回给调用方，不 panic。

## 6. 并发细节

- **goroutine 边界**：gin 本身不 spawn goroutine，由 `net/http` 每连接一个 goroutine 调 `ServeHTTP`；引擎对象在服务期间只读（trees 构建完成后不再改）。
- **Context 池化**：`pool sync.Pool`（`gin.go:183`）复用 Context，避免每请求分配；`pool.New` 调 `allocateContext(maxParams)` 预分配 Params 与 skippedNodes 容量（`gin.go:229`、`gin.go:252`）。归还前 `reset()` 把切片长度归零但保留底层容量（见 `context.go:103`）。
- **sync.Once 一次性收尾**：`routeTreesUpdated` 保证 `updateRouteTrees` 并发下只跑一次（`gin.go:97`、`gin.go:663`）；注册期允许 `addRoute` 写入 `\:` 转义形式，首个请求统一还原。
- **注册期非并发安全**：`addRoute` 直接改树结构，注释明确"Not concurrency-safe"（见 `tree.go:134`）；约定所有 `GET/POST...` 注册在 `Run` 之前完成，运行期只读。
- **context.Context 传播**：`Context` 自身在 `ContextWithFallback` 下实现 `context.Context`（见 `context.go:1502-1544`），请求取消/超时经由 `c.Request.Context()` 向下游传播。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `gin.go` 全文：Engine 构建/启动/ServeHTTP/派发/重定向/可信代理。
- `router-group`（注册）与 `radix-tree`（树结构与 getValue）的调用契约（分别见对应叶子）。
- `response_writer` 的 `reset/WriteHeaderNow` 调用。

**Out-of-Scope（不在本仓库源码内）**
- `net/http` 服务器与连接管理、`http.Server.ListenAndServe`（Go 标准库）。
- `golang.org/x/net/http2/h2c`（H2C 明文 HTTP/2）、`github.com/quic-go/quic-go/http3`（QUIC/HTTP3）——外部依赖。
- OS socket / 文件监听 / 端口绑定（操作系统与 Go 标准库网络栈）。

**不做什么**：不解析路由树内部匹配细节、不定义 Context 字段语义、不实现中间件（Logger/Recovery 见 `middleware` 相关文件）。

## 8. 与相邻子系统交互

- **net/http → Engine**：每请求调 `ServeHTTP(w, req)`，Engine 实现 `http.Handler`。
- **Engine → router-group（注册期）**：`addRoute` ← `RouterGroup.handle` 调用，把合并后的 handler 链交给 `engine.trees`。
- **Engine → radix-tree（运行期）**：`root.getValue(...)` 完成查表，回填 `c.Params`/`c.handlers`/`c.fullPath`。
- **Engine → context-object**：池化取出的 Context 上 `reset()` 后执行 `c.Next()`；派发失败走 `serveError`/`redirectRequest`。
- **Engine → response-writer**：`c.writermem.reset(w)` 包装底层 `http.ResponseWriter`，结束时 `WriteHeaderNow()`。
- **ClientIP → Engine 可信代理**：`Context.ClientIP()` 反向调用 `engine.isTrustedProxy/validateHeader`。

## 9. 语言专项适配口径

- **并发模型**：典型的"库形态 + 外部事件驱动"——无自有 goroutine，靠 `net/http` 并发调 `ServeHTTP`；并发安全靠 `sync.Pool`（Context 复用）+ `sync.Once`（树收尾），运行期对 trees 只读、注册期写，二者生命周期由"先注册后 Run"的约定切分。
- **非 K8s 控制器**：无 Reconcile/informer/workqueue，故不做事件管道与最终一致性分析；`sync.Once` 在此是"一次性初始化"而非 leader election。
- **单二进制库形态**：gin 无 `cmd/` 入口，本身是被 `import` 的库，使用者在自己的 `main` 里 `gin.New()`/`Run()`；`Run*` 只是便捷封装 `http.Server`。
- **internal 边界**：`Engine` 直接引用 `internal/bytesconv`、`internal/fs`（模板解析用），符合 Go `internal` 仅本仓库可导入的约束；不跨包暴露内部细节。
- **context.Context 传播**：请求级取消经 `c.Request.Context()` 传播，`ContextWithFallback` 决定 Context 是否对外当作 `context.Context` 使用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 引擎组件与边界架构图 | `engine-lifecycle-architecture.html` | architecture | showcase |
| 请求派发主路径时序图 | `engine-lifecycle-sequence.html` | sequence | showcase |

- JSON IR 源文件位于 `json/` 目录（`engine-lifecycle-architecture.json`、`engine-lifecycle-sequence.json`）。
- 时序图只画"命中路径"主链路（net/http → ServeHTTP → handleHTTPRequest → getValue → Next 中间件/业务 → WriteHeaderNow → 归还池），404/405/重定向分支在第 3 节文字化展开。
