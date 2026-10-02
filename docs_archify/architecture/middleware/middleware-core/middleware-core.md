# 中间件链执行协议与静态文件服务（middleware-core）

> 本文是 `middleware` 域下的叶子子系统文档。域级总览见 [`../middleware.md`](../middleware.md)。
> gin 顶层**没有独立的 middleware.go**：中间件的"注册—组合—链式执行"协议分散在 `gin.go`、
> `routergroup.go`、`context.go`，静态文件服务在 `fs.go` + `routergroup.go`。本文只展开这套
> 执行协议本身；日志中间件见 [`../logger/logger.md`](../logger/logger.md)，崩溃恢复与 BasicAuth
> 见 [`../recovery-auth/recovery-auth.md`](../recovery-auth/recovery-auth.md)。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 处理器类型定义 | `HandlerFunc` 是 gin 一切中间件/业务处理器的统一签名 `func(*Context)`；`HandlersChain` 是其切片，`Last()` 取末位主处理器 | `gin.go:51`、`gin.go:57`、`gin.go:60` |
| 中间件注册 | `Engine.Use()` 把全局中间件追加到根 `RouterGroup.Handlers`，并重建 404/405 链 | `gin.go:340`、`routergroup.go:65` |
| 路由分组继承 | `RouterGroup.Group()` 子组通过 `combineHandlers` 继承父组中间件前缀 | `routergroup.go:72`、`routergroup.go:251` |
| 路由注册时链组合 | `RouterGroup.handle()` 把"组中间件 + 路由级处理器"合并成最终 `HandlersChain` 后插入基数树 | `routergroup.go:86`、`gin.go:364` |
| 链式执行引擎 | `Context.Next()` 用 `int8` 游标顺序执行链；`Abort()` 把游标打到 `abortIndex` 截断后续处理器 | `context.go:198`、`context.go:217`、`context.go:57` |
| 上下文池 | `Engine.ServeHTTP` 从 `sync.Pool` 取/还 `Context`，`reset()` 清理游标与缓存 | `gin.go:662`、`context.go:103` |
| 请求分派 | `handleHTTPRequest` 基数树命中后把 `value.handlers` 赋给 `c.handlers` 并 `c.Next()` | `gin.go:690`、`gin.go:719` |
| 静态目录服务 | `Static/StaticFS` 注册 `/p/*filepath`，用 `http.StripPrefix`+`http.FileServer` serve，可禁目录列表 | `routergroup.go:206`、`routergroup.go:213`、`routergroup.go:226` |
| 静态单文件 | `StaticFile/StaticFileFS` 注册单路由，分别走 `c.File`/`c.FileFromFS` | `routergroup.go:175`、`routergroup.go:184`、`context.go:1341`、`context.go:1346` |
| 目录列表开关文件系统 | `Dir(root,listDirectory)` 不允许列表时包成 `OnlyFilesFS`，其 `Readdir` 恒返回 nil | `fs.go:42`、`fs.go:13`、`fs.go:33` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `HandlerFunc` | `gin.go:51` | 中间件与处理器统一签名；gin 没有独立的 `Middleware` 类型，中间件就是普通 HandlerFunc，区别仅在于是否在内部调用 `c.Next()` |
| `HandlersChain` | `gin.go:57` | `[]HandlerFunc`，一条执行链；`Last()` 取末位（主业务处理器） |
| `RouterGroup` | `routergroup.go:55` | 路由组：持有 `Handlers HandlersChain`（本组中间件前缀）、`basePath`、回指 `engine`、`root` 标记；`Engine` 内嵌它（`gin.go:93`），故 Engine 本身就是根组 |
| `Engine` | `gin.go:92` | 框架实例：内嵌 RouterGroup，持有路由树 `trees`、`sync.Pool` 上下文池、404/405 链（`allNoRoute/allNoMethod`）与一堆路由开关 |
| `Context.index int8` | `context.go:68` | **链游标**，是整套执行协议的核心状态；`abortIndex=math.MaxInt8>>1`（`context.go:57`）作为"已中止"哨兵 |
| `combineHandlers` | `routergroup.go:251` | 注册期合并：先拷 `group.Handlers` 再拷路由处理器；`assert1(finalSize < abortIndex)` 限制链长 |
| `OnlyFilesFS` / `neutralizedReaddirFile` | `fs.go:13` / `fs.go:28` | 包装 `http.FileSystem`，把 `Readdir` 覆写为恒 nil，从而关闭 `http.FileServer` 的目录列表 |

## 3. 关键调用链

### 3.1 注册期：中间件如何进入一条路由的最终链

1. 用户调用 `engine.Use(Logger(), Recovery())` → `Engine.Use`（`gin.go:340`）转发到 `RouterGroup.Use`（`routergroup.go:65`），把中间件 append 到根组 `group.Handlers`，随后 `rebuild404Handlers()`/`rebuild405Handlers()`（`gin.go:356`、`gin.go:360`）把全局中间件也并进 404/405 链。
2. 用户调用 `r.GET("/users", h)` → `RouterGroup.handle`（`routergroup.go:86`）：先 `calculateAbsolutePath`，再 `combineHandlers(handlers)`（`routergroup.go:251`）——`copy(merged, group.Handlers)` 拷入组中间件，再拷入路由级处理器，得到 `[全局中间件..., 组中间件..., 路由处理器]`。
3. 合并后的 `HandlersChain` 交给 `Engine.addRoute`（`gin.go:364`），断言路径以 `/` 开头、方法非空、至少一个处理器，随后插入方法基数树 `root.addRoute(path, handlers)`（`tree.go:135`）。**中间件在此刻就被"烘焙"进路由节点**，请求期不再做拼接。

### 3.2 请求期：链如何按序执行（含 Next/Abort 语义）

1. `net/http` 调用 `engine.ServeHTTP(w, req)`（`gin.go:662`）：`routeTreesUpdated.Do` 一次性构建路由树，从 `engine.pool` 取一个 `Context`，`c.writermem.reset(w)` + `c.reset()`（`context.go:103`，把 `index` 置回 `-1`、`handlers=nil`）。
2. `handleHTTPRequest`（`gin.go:690`）在基数树 `getValue` 命中后：`c.handlers = value.handlers`（`gin.go:720`），然后 `c.Next()`（`gin.go:722`）启动链式执行。
3. `c.Next()`（`context.go:198`）：`c.index++` 后进入 `for c.index < safeInt8(len(c.handlers))` 循环，逐个调用 `c.handlers[c.index](c)` 并 `index++`。中间件若想"放行后续处理器"，就在自身内部调用 `c.Next()`——这会在当前栈帧内同步执行完下游整条链，再返回继续中间件的后置逻辑（这就是 Logger 能计时、Recovery 能 defer recover 的原理）。
4. 任一处理器调用 `c.Abort()`（`context.go:217`）把 `c.index = abortIndex`（`context.go:57`），外层 `Next` 循环条件 `c.index < len(handlers)` 立即不成立，**后续处理器被跳过，但当前处理器继续跑完**（`AbortWithStatus` `context.go:223` 会先写状态码再 Abort）。`IsAborted()`（`context.go:209`）用 `c.index >= abortIndex` 判断。
5. 链跑完后 `c.writermem.WriteHeaderNow()`（`gin.go:723`）确保状态码落盘，`ServeHTTP` 把 `c` 放回 pool。

### 3.3 静态文件服务链路

1. `r.Static("/static", "/var/www")`（`routergroup.go:206`）→ `StaticFS`（`routergroup.go:213`）断言路径不含 `:`/`*`，构造 `Dir("/var/www", false)`（`fs.go:42`，因 `listDirectory=false` 返回 `OnlyFilesFS`），注册 `GET/HEAD /static/*filepath`。
2. 命中后进入 `createStaticHandler` 返回的闭包（`routergroup.go:230`）：若 `fs` 是 `*OnlyFilesFS` 先写 404 头（禁目录列表）；取 `c.Param("filepath")` 并 `fs.Open` 预探文件存在性，不存在则把 `c.handlers` 切到 `engine.noRoute`、`c.index=-1` 后 return（交回框架 404）；存在则 `f.Close()` 后交给 `http.StripPrefix(absolutePath, http.FileServer(fs)).ServeHTTP(...)`。
3. 单文件路由 `StaticFile`（`routergroup.go:175`）直接在处理器里 `c.File(filepath)`（`context.go:1341`）→ `http.ServeFile`；`StaticFileFS`（`routergroup.go:184`）走 `c.FileFromFS`（`context.go:1346`），后者临时改写 `c.Request.URL.Path` 再调用 `http.FileServer`，并用 defer 恢复原路径。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|---|---|---|
| `Engine.RedirectTrailingSlash` | `New()` 默认 `true`：尾斜杠不匹配时 301/307 重定向 | `gin.go:104`、`gin.go:211`、`gin.go:727` |
| `Engine.RedirectFixedPath` | 默认 `false`：清理 `../`、`//` 后大小写不敏感重定向 | `gin.go:115`、`gin.go:212`、`gin.go:731` |
| `Engine.HandleMethodNotAllowed` | 默认 `false`：开启后路径命中但方法不对时走 405 链并写 `Allow` 头 | `gin.go:123`、`gin.go:213`、`gin.go:738` |
| `Engine.UseH2C` | 默认 `false`；`Handler()` 为 true 时包一层 `http2.Server` h2c 升级器 | `gin.go:170`、`gin.go:244` |
| `Default()` 预置中间件 | `New()` 后 `engine.Use(Logger(), Recovery())` | `gin.go:236` |
| `Static(relative, root)` 的 `listDirectory` | 硬编码 `Dir(root, false)`，默认**关闭**目录列表 | `routergroup.go:207`、`fs.go:42` |
| 链长上限 | `combineHandlers` 断言 `finalSize < abortIndex`（≈63），超限 panic "too many handlers" | `routergroup.go:253`、`context.go:57` |

## 5. 错误与重试语义

- **注册期错误是编程错误，直接 panic**：`addRoute` 用 `assert1`（`utils.go:86`）在路径非 `/` 开头、方法为空、处理器为空时 panic（`gin.go:365-367`）；静态路径含 `:`/`*` 时 panic（`routergroup.go:191`、`routergroup.go:214`）。这类错误不会重试，属于启动期配置错误。
- **请求期 404/405 不报错**：`handleHTTPRequest` 未命中时把 `c.handlers` 切到 `engine.allNoRoute`（`gin.go:758`）并 `serveError`（`gin.go:764`）写默认 404 文本；方法不允许时切 `allNoMethod` 并写 `Allow` 头（`gin.go:751`）。两者都是正常控制流，不进入 `c.Errors`。
- **静态文件不存在**：`createStaticHandler` 探测 `fs.Open` 失败时不 panic，而是切到 `engine.noRoute` 并重置 `c.index=-1`（`routergroup.go:239-243`），把后续交回框架 404 链处理。
- **无重试/退避**：本叶子是同步内存路由与文件服务，不涉及网络 IO 重试；唯一的"恢复"是 Recovery 中间件（见 recovery-auth 叶子）在 `c.Next()` 内部 defer recover。

## 6. 并发细节

- **Context 复用是并发安全的关键**：`Engine.pool sync.Pool`（`gin.go:183`）在 `ServeHTTP` 里 `Get()`/`Put()`（`gin.go:667`、`gin.go:674`），每个请求独占一个 `Context` 实例；`reset()`（`context.go:103`）在归还前清空 `handlers/index/Keys/Errors/queryCache` 等。**goroutine 中使用必须先 `c.Copy()`**（`context.go:122`），拷贝时 `index` 直接置为 `abortIndex` 以阻止副本继续跑主链。
- **路由树并发只读**：路由在启动期注册；`routeTreesUpdated sync.Once`（`gin.go:97`）保证 `updateRouteTrees` 只在首次请求时执行一次，之后 `trees` 只读，请求期无锁读。
- **`Context.Keys` 用 `sync.RWMutex` 保护**（`context.go:76`）：`Set` 写锁、`Get` 读锁（`context.go:286`、`context.go:298`），支持中间件间传值（如 BasicAuth 写入 `AuthUserKey`）。
- **`c.Next()` 是同步栈式调用**，不创建 goroutine；中间件内若起 goroutine 必须 `c.Copy()`，否则复用会 data race。
- **无 context.Context 超时传播**：gin 的 `c.Next()` 游标机制与 Go 标准 `context.Context` 无关（`ContextWithFallback` 开关见 `gin.go:172`，属另一叶子范围）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `gin.go`：`HandlerFunc/HandlersChain` 定义、`Engine.Use/addRoute/ServeHTTP/handleHTTPRequest`、上下文池。
- `routergroup.go`：`RouterGroup.Use/Group/handle/combineHandlers` 与静态文件注册（`Static/StaticFS/StaticFile/StaticFileFS/createStaticHandler/staticFileHandler`）。
- `context.go`：`Next/Abort/IsAborted/AbortWithStatus/reset/File/FileFromFS`。
- `fs.go`：`Dir/OnlyFilesFS/neutralizedReaddirFile`。
- `utils.go`：`joinPaths/lastChar/assert1` 等路径与断言辅助。

**Out-of-Scope（不在本仓库源码内）**
- 路由基数树的具体匹配算法（前缀树插入/`getValue`/TSR 判断）在 `tree.go`，属"路由"域，本文只引用其输出 `value.handlers`。
- `net/http` 标准库：`http.FileServer`、`http.ServeFile`、`http.StripPrefix`、`http.FileSystem` 均为 Go 标准库，不在本仓库源码内。
- 业务处理器（用户注册的末位 handler）与 Logger/Recovery/BasicAuth 中间件分别见各自叶子。
- 错误聚合 `c.Errors` 的类型化过滤见 support 域 errors 叶子；中间件如何写日志/如何 recover 见本域 logger、recovery-auth 叶子。

## 8. 与相邻子系统交互

- **上游**：`net/http` 服务器（外部）→ `Engine.ServeHTTP`（`gin.go:662`）。
- **注册期**：用户 API（`r.Use`/`r.Group`/`r.GET`）→ `RouterGroup.handle` → `Engine.addRoute` → 路由树（路由域 `tree.go`）。
- **请求期**：`handleHTTPRequest` 命中树 → 取出 `HandlersChain` → `Context.Next()` 依次驱动中间件与业务处理器；链中任一处理器可 `c.Abort()` 截断。
- **下游交互**：Logger（`logger.go`）与 Recovery（`recovery.go`）正是靠"在处理器内部先 `c.Next()`"插入到链首，从而包住整条下游链；BasicAuth（`auth.go`）在链中判定失败后 `AbortWithStatus(401)`。
- **静态文件**：`createStaticHandler` → 标准库 `http.FileServer` → 磁盘文件系统（外部 OS）。

## 9. 语言专项适配口径（Go）

- **并发模型**：gin 是**同步内联中间件模型**，不是 K8s 式 Reconcile/informer 控制器——没有 workqueue、没有 leader election、没有 watch 管道。并发安全完全依赖 `sync.Pool` 的 Context 每请求独占 + `sync.Once` 的路由树一次性构建 + `Context.mu` 保护 `Keys` map。这是 Web 框架与平台控制器的本质差异，特此说明。
- **`context.Context` 传递**：gin 自有 `Context` 类型（`context.go:61`）包装 `*http.Request`，不使用 `context.Context` 的 Done/Deadline 作为链控制机制；链控制是自研 `index int8` 游标。异步派生 goroutine 必须 `c.Copy()`（`context.go:122`），否则 pool 复用导致竞态。
- **internal 边界**：本叶子只 import 了标准库；`internal/bytesconv` 由 recovery/auth 使用（见 recovery-auth、internal-utils 叶子），本叶子不直接依赖。
- **多二进制**：项目无 `cmd/`，是纯库；`gin.Engine` 本身实现 `http.Handler` 接口（`gin.go:662`），由用户用 `http.Serve(engine)` 启动。
- **接口定义位置**：`IRoutes/IRouter`（`routergroup.go:33`、`routergroup.go:27`）定义在消费侧（框架对外 API 面），由 `RouterGroup` 实现——这是面向接口的路由 DSL。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 中间件链组合与执行架构图 | [middleware-core-architecture.html](middleware-core-architecture.html) | architecture | showcase |
| 请求链式执行时序图 | [middleware-core-sequence.html](middleware-core-sequence.html) | sequence | showcase |
| 链游标生命周期图 | [middleware-core-lifecycle.html](middleware-core-lifecycle.html) | lifecycle | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 workflow 图——中间件链的"分步流程"本质是单次请求的消息交互，已由 sequence 图表达；未生成 dataflow 图——本叶子无"数据从源到目的地的变换管道"语义（日志采集见 logger 叶子），按资源节省原则省略。
