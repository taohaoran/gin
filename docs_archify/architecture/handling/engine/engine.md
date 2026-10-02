# 引擎核心（engine）

> 本文是 `handling` 域下的叶子子系统文档。域级总览见 `../handling.md`。
> 本文只展开 **Engine 实例的构建、运行入口、请求分发与可信代理校验**，不重复展开路由基数树（见 routing 域）、中间件链执行（见 context 叶子）、响应写入（见 response-writer 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`，Go 1.26.0，版本 `v1.12.0`（`version.go:8`）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Engine 构造 | `New()` 空白实例（无中间件）、`Default()` 自带 Logger+Recovery；OptionFunc 模式后配置 | `gin.go:202`（New）、`gin.go:236`（Default）、`gin.go:348`（With） |
| 运行入口 Run 家族 | `Run`/`RunTLS`/`RunUnix`/`RunFd`/`RunListener`/`RunQUIC` 六种监听形态，均阻塞调用 goroutine | `gin.go:540/561/581/607/645/630` |
| http.Handler 适配 | `ServeHTTP` 满足 `net/http.Handler` 接口；`Handler()` 在开启 H2C 时包一层 `h2c.NewHandler` | `gin.go:662`、`gin.go:243` |
| 请求分发 | `handleHTTPRequest`：按 method 取基数树根 → `getValue` 查路由 → 命中则执行 handler 链；未命中则尾随斜杠/固定路径重定向 → 405 → 404 | `gin.go:690` |
| Context 对象池 | `sync.Pool` 复用 `*Context`，`pool.New=allocateContext` 预分配 params/skippedNodes 容量 | `gin.go:183`、`gin.go:252`、`gin.go:667` |
| 全局中间件挂载 | `Use()` 挂到根 RouterGroup，并重建 404/405 处理器链 | `gin.go:340`、`gin.go:356/360` |
| 路由注册收口 | `addRoute` 校验 path/method/handlers，按 method 挂到基数树，并统计 maxParams/maxSections | `gin.go:364` |
| 可信代理 | `SetTrustedProxies` 解析 CIDR 白名单；`validateHeader` 反向解析 X-Forwarded-For；`isUnsafeTrustedProxies` 启动告警 | `gin.go:451/414/482/457` |
| HTML 模板加载 | `LoadHTMLGlob`/`LoadHTMLFiles`/`LoadHTMLFS`/`SetHTMLTemplate`；debug 模式走 `render.HTMLDebug` 热重载，生产走 `render.HTMLProduction` | `gin.go:272/288/300/312` |
| 模式与调试 | `SetMode`/`IsDebugging`（debug/release/test）；`debugPrint*` 系列仅 debug 模式输出；`DefaultWriter`/`DefaultErrorWriter` | `mode.go:58/22`、`debug.go:22/56` |
| 重定向辅助 | `redirectTrailingSlash`/`redirectFixedPath`/`redirectRequest`（GET 301、其余 307）；`sanitizePathChars` 过滤 X-Forwarded-Prefix | `gin.go:781/808/820/799` |
| 404/405 兜底 | `serveError` 写纯文本错误体；`NoRoute`/`NoMethod` 用户自定义 | `gin.go:764/326/332` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Engine` struct | `gin.go:92` | 框架实例：内嵌 `RouterGroup`，持路由树 `trees methodTrees`、对象池 `pool`、HTMLRender、可信代理 CIDR 表、全部路由开关 |
| `HandlerFunc` | `gin.go:51` | 中间件/处理器统一签名 `func(*Context)` |
| `HandlersChain` | `gin.go:57` | `[]HandlerFunc`，`Last()` 取主处理器 |
| `OptionFunc` | `gin.go:54` | 函数式配置 `func(*Engine)`，`With` 依次执行 |
| `RouterGroup`（内嵌） | `routergroup.go` | 路由分组与中间件组合；`Engine` 自身 root 组 `basePath="/", root=true` |
| `methodTrees` | `tree.go` | 按 HTTP method 分棵的基数树数组 |
| `RouteInfo`/`RoutesInfo` | `gin.go:68/76` | `Routes()` 自省输出（method/path/handler） |
| `sync.Once routeTreesUpdated` | `gin.go:97` | 惰性把 `\: ` 转义还原为 `:`，仅在首次 `ServeHTTP` 时执行一次 |
| `defaultTrustedCIDRs` | `gin.go:39` | 默认 `0.0.0.0/0` + `::/0`（信任全部，启动告警） |
| 平台常量 | `gin.go:82-87` | `PlatformGoogleAppEngine`/`PlatformCloudflare`/`PlatformFlyIO`，对应各自客户端 IP 头 |

## 3. 关键调用链

### 调用链 A：HTTP 请求主路径（`Engine.ServeHTTP`）
1. `net/http.Server` 收到连接，为每请求起 goroutine，回调 `Engine.ServeHTTP(w, req)`（`gin.go:662`）。
2. `routeTreesUpdated.Do(...)`（`gin.go:663`）首次调用时把路由树里的 `\:` 还原为 `:`，之后空转。
3. `engine.pool.Get().(*Context)`（`gin.go:667`）从 `sync.Pool` 取一个复用 Context；`c.writermem.reset(w)` 绑定底层 `http.ResponseWriter`，`c.reset()` 清状态（详见 context 叶子）。
4. 进入 `handleHTTPRequest(c)`（`gin.go:690`）：按 `c.Request.Method` 线性扫 `engine.trees` 找同 method 树根（`gin.go:708-713`）。
5. `root.getValue(rPath, c.params, c.skippedNodes, unescape)`（`gin.go:715`）在基数树中查路径并填充路径参数。
6. 命中 handler：`c.handlers = value.handlers` → `c.Next()`（`gin.go:722`）顺序执行中间件链 → `c.writermem.WriteHeaderNow()` 兜底落头 → `pool.Put(c)` 回收。
7. 未命中且 `value.tsr && RedirectTrailingSlash` → `redirectTrailingSlash(c)`（`gin.go:728`）；`RedirectFixedPath` 开 → `redirectFixedPath`（`gin.go:731`）。
8. 仍未命中：`HandleMethodNotAllowed` 开则遍历其余 method 树拼 `Allow` 头后 `serveError(405)`（`gin.go:738-755`）；否则 `c.handlers = engine.allNoRoute` → `serveError(404)`（`gin.go:758-759`）。

### 调用链 B：启动监听（`Run` 家族）
1. `engine.Run(addr...)`（`gin.go:540`）：先 `isUnsafeTrustedProxies()` 打印告警（`gin.go:543`），`updateRouteTrees()`（`gin.go:547`）还原转义冒号，`resolveAddress(addr)`（`gin.go:548`，`utils.go:148`）决定监听地址（无参则读 `$PORT`，缺省 `:8080`）。
2. 构造 `&http.Server{Addr, Handler: engine.Handler()}`（`gin.go:550`）；`Handler()` 在 `UseH2C` 时用 `h2c.NewHandler` 包一层（`gin.go:243-249`）。
3. `server.ListenAndServe()`（`gin.go:554`）阻塞。`RunTLS`/`RunUnix`/`RunFd`/`RunListener`/`RunQUIC` 同构，仅监听构造不同（TLS 证书 / unix socket / fd → `net.FileListener` / 外部 listener / `quic-go` 的 `http3.ListenAndServeQUIC`）。

### 调用链 C：可信代理 → 客户端 IP
1. 用户 `SetTrustedProxies([]string{"10.0.0.0/8"})`（`gin.go:451`）→ `parseTrustedProxies()`（`gin.go:462`）→ `prepareTrustedCIDRs()`（`gin.go:414`）：无 `/` 的裸 IP 补 `/32` 或 `/128`，再 `net.ParseCIDR`。
2. 请求处理时 `Context.ClientIP()`（见 context 叶子）回调 `engine.validateHeader(header)`（`gin.go:482`）：从 `X-Forwarded-For` 末尾反向遍历，遇到首个不在 `trustedCIDRs` 的 IP 即视为真实客户端。
3. 若 `trustedCIDRs` 含 `0.0.0.0/0` 或 `::/0`，`isUnsafeTrustedProxies()` 返回 true（`gin.go:457`），所有 `Run*` 启动时打印安全警告。

## 4. 配置项

| 配置项 | 默认值 | 行为 | 位置 |
|---|---|---|---|
| `RedirectTrailingSlash` | true | 路径尾斜杠不匹配时 301/307 重定向 | `gin.go:104`、`gin.go:211` |
| `RedirectFixedPath` | false | 清理 `//`/`../` 后大小写不敏感查找并重定向 | `gin.go:115`、`gin.go:212` |
| `HandleMethodNotAllowed` | false | 路径命中但 method 不匹配时返回 405 + Allow 头 | `gin.go:123`、`gin.go:213` |
| `ForwardedByClientIP` | true | 从 `RemoteIPHeaders` 解析客户端 IP | `gin.go:129`、`gin.go:214` |
| `RemoteIPHeaders` | `["X-Forwarded-For","X-Real-IP"]` | 按序尝试的客户端 IP 头 | `gin.go:159`、`gin.go:215` |
| `TrustedPlatform` | `""`（`defaultPlatform`） | 设为 `Platform*` 常量时改信该平台头 | `gin.go:163`、`gin.go:216` |
| `UseRawPath` | false | 用 `URL.RawPath` 查参数 | `gin.go:140`、`gin.go:217` |
| `UseEscapedPath` | false | 用 `URL.EscapedPath()` 查参数，覆盖 RawPath | `gin.go:144`、`gin.go:218` |
| `UnescapePathValues` | true | 路径参数是否反转义 | `gin.go:149`、`gin.go:220` |
| `RemoveExtraSlash` | false | 额外斜杠时 `cleanPath` 后再查 | `gin.go:153`、`gin.go:219` |
| `MaxMultipartMemory` | 32MB | `ParseMultipartForm` 上限 | `gin.go:167`、`gin.go:221`、`gin.go:26` |
| `UseH2C` | false | `Handler()` 包 h2c cleartext HTTP/2 | `gin.go:170`、`gin.go:243` |
| `ContextWithFallback` | false | 开启 `Context.Value/Deadline/Done/Err` 回退 | `gin.go:173` |
| `trustedProxies` | `["0.0.0.0/0","::/0"]` | 默认信任全部代理（不安全，启动告警） | `gin.go:225` |
| `GIN_MODE` 环境变量 | debug（测试构建自动 test） | `SetMode` 接受 debug/release/test，未知值 panic | `mode.go:17/58` |
| `$PORT` | 缺省 `:8080` | `Run()` 无参时使用 | `utils.go:148` |
| 模板分隔符 | `{{ }}` | `Delims(left,right)` 自定义 | `gin.go:175/259` |
| `SecureJsonPrefix` | `while(1);` | `SecureJSON` 前缀防 JSON 劫持 | `gin.go:176/265` |

## 5. 错误与重试语义

- **路由未命中**：不返回错误，而是走兜底链——`redirectTrailingSlash`/`redirectFixedPath` 发 3xx；`HandleMethodNotAllowed` 命中其他 method 则 405；否则 404。`serveError`（`gin.go:764`）先 `c.Next()` 让用户 NoRoute/NoMethod 中间件有机会写响应，若已 Written 则直接返回，否则写 `text/plain` 默认错误体。
- **`SetTrustedProxies`**：`prepareTrustedCIDRs` 解析失败时返回 `*net.ParseError`，调用方（通常 `init`/启动期）决定是否终止；运行期不再重试。
- **`SetMode` 未知值**：直接 `panic("gin mode unknown: ...")`（`mode.go:75`），启动期快速失败。
- **`LoadHTMLGlob`/`LoadHTMLFiles`**：模板解析失败用 `template.Must` panic（`gin.go:275/294`），启动期快速失败。
- **`Run*` 家族**：监听/服务错误原样返回给调用方，`defer debugPrintError(err)` 在 debug 模式打印；不自动重试（HTTP server 语义，重启由调用方负责）。
- **`resolveAddress` 多参**：`panic("too many parameters")`（`utils.go:160`）。
- 本叶子**无重试队列、无退避**；错误要么启动期 panic/返回，要么转为 HTTP 状态码。

## 6. 并发细节

- **`Engine` 本身在 `Run` 之后只读**：路由树 `trees`、`trustedCIDRs`、`HTMLRender` 在启动期构建完成；运行期 `ServeHTTP` 只读访问。`SetHTMLTemplate` 明确标注**非线程安全**，要求在监听前调用（`debug.go:100`）。
- **`routeTreesUpdated sync.Once`**（`gin.go:97`）：保证多 goroutine 首次并发 `ServeHTTP` 时 `updateRouteTrees` 只执行一次。
- **`pool sync.Pool`**（`gin.go:183`）：每请求 Get/Put `*Context`，避免 Context 及其 params 切片反复分配；`pool.New=allocateContext(maxParams)`（`gin.go:229`）按已注册路由的最大参数数预分配容量。Pool 由 runtime 管理 GC 回收，无显式关闭。
- **goroutine 边界**：`Run*` 在调用方 goroutine 阻塞；每请求的业务 handler 由 `net/http.Server` 起新 goroutine 执行，`Engine` 自身不起后台 goroutine。
- **`ginMode int32` + `atomic.Value modeName`**（`mode.go:48-50`）：`IsDebugging` 用 `atomic.LoadInt32` 无锁读，支持运行期 `SetMode`。
- **无共享可变请求状态**：每请求独立 Context，请求间不共享；`Context` 字段在 `pool.Get` 后由 `c.reset()` 清零。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `gin.go`：Engine 结构体、New/Default、Run 家族、ServeHTTP/handleHTTPRequest、trusted proxies、HTML 加载、错误兜底与重定向
- `mode.go`：debug/release/test 模式与全局开关
- `debug.go`：debug 打印、版本检查、模板加载日志
- `version.go`：版本常量
- 与 `routergroup.go`（内嵌 RouterGroup）、`tree.go`（基数树 getValue）、`context.go`（Context）、`response_writer.go`（writermem）、`render/`（HTMLRender）同包协作

**Out-of-Scope（不在本仓库源码内）**
- `net/http.Server` / `net.Listen` / `http.Redirect`：Go 标准库，仅作为外部依赖调用
- `quic-go/http3`（`RunQUIC`）、`golang.org/x/net/http2`、`h2c`：第三方依赖，不在本仓库源码内
- `html/template`：Go 标准库模板引擎
- 中间件具体实现（Logger/Recovery）在 `logger.go`/`recovery.go`，属 middleware 域
- 路由基数树的插入/查找算法细节在 `tree.go`，属 routing 域
- Context 的参数取值/绑定/渲染方法在 `context.go`，属本域 context 叶子

## 8. 与相邻子系统交互

- **上游**：用户代码 `gin.Default()`/`engine.Run()`；Go 标准库 `http.Server` 把每个请求作为 `http.Handler` 回调进来。
- **下游 → routing 域**：`handleHTTPRequest` 调 `methodTree.root.getValue(rPath, ...)`（`gin.go:715`）查路由并填充 `c.params`；`addRoute` 调 `root.addRoute` 注册。
- **下游 → context 叶子**：从 `pool` 取 `*Context`，设置 `c.handlers` 后调 `c.Next()` 驱动中间件链；重定向/错误也通过 `c.writermem` 写响应。
- **下游 → response-writer 叶子**：`c.writermem.reset(w)` 包装底层 `http.ResponseWriter`；`WriteHeaderNow()` 在 handler 链结束时兜底落头。
- **下游 → render 子包**：`HTMLRender` 字段被 `Context.HTML` 调用（见 context 叶子）；Engine 只负责加载模板与选择 HTMLDebug/HTMLProduction 实现。
- **下游 → binding 子包**：`mode.go` 的 `DisableBindValidation`/`EnableJsonDecoder*` 是对 `binding.Validator` 全局开关的转发。

## 9. 语言专项适配口径（Go）

- **并发模型**：gin 是典型的「每请求一 goroutine」HTTP 服务模型，**不是 K8s 控制器模式**——没有 Reconcile/informer/workqueue。并发原语只用了两个：`sync.Pool`（Context 复用，`gin.go:183`）与 `sync.Once`（路由树转义还原只跑一次，`gin.go:97`）。模式开关用 `atomic.Int32` + `atomic.Value` 无锁读（`mode.go:48`）。
- **context.Context 传递**：本叶子不直接管理 `context.Context` 超时/取消；`c.Request.Context()` 的继承与 `ContextWithFallback` 语义在 context 叶子展开。
- **多二进制**：本仓库是纯库，**无 `cmd/` 入口**（facts.md 已确认）；`Run*` 是给用户 main 包调用的阻塞式快捷方法，不是独立二进制。`ginS/` 为示例全局服务器，仅边界提及。
- **internal 边界**：`internal/bytesconv`（零拷贝 string↔[]byte）、`internal/fs`（文件系统适配）按 Go internal 规则仅包内可用；依赖方向为 `gin` 根包 → `internal/*`、`render/`、`binding/`，无环。
- **接口定义位置**：`render.HTMLRender` 接口在 `render/` 子包定义，由 Engine 持有其实现（`HTMLDebug`/`HTMLProduction`），依赖倒置——Engine 面向 render 包接口编程。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| 引擎架构图 | `engine-architecture.html` | architecture | showcase（一次通过） |
| 请求分发时序图 | `engine-sequence.html` | sequence | showcase（初次 y 坐标下限 160、participant 标签过宽两轮修复后通过） |

- 本叶子未生成 dataflow 图：Engine 主路径是同步请求分发（控制流），数据管道语义弱；路由匹配→handler 链的数据变换已由 context 叶子的 dataflow 覆盖，按资源节省原则省略。
- 本叶子未生成 workflow/lifecycle 图：无多泳道审批流程；Engine 本身无状态机（路由树为静态只读结构）。
- JSON IR 位于 `json/` 目录。
