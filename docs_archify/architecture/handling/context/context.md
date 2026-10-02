# 请求上下文（context）

> 本文是 `handling` 域下的叶子子系统文档。域级总览见 `../handling.md`。
> 本文只展开 **`Context` 的链式执行协议、参数/表单取值、绑定与渲染快捷入口、客户端 IP 与 context.Context 适配**，不重复展开 Engine 分发（见 engine 叶子）、responseWriter 写入（见 response-writer 叶子）、binding/render 子包内部算法。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 链式执行协议 | `Next()` 顺序推进 `handlers`；`Abort()` 把 index 跳到 `abortIndex` 阻断后续；`AbortWithStatus*`/`AbortWithError` 组合中断+写响应 | `context.go:198/217/223/232/240/248` |
| 跨中间件键值 | `Set/Get/MustGet` 及 `GetString/GetInt/GetBool...` 全套类型化取值（泛型 `getTyped`）；`Delete` | `context.go:286/298/306/313/492` |
| 路径参数 | `Param`、`AddParam`（测试用）；`Params` 由路由树填充 | `context.go:513/522` |
| Query 取值 | `Query/DefaultQuery/GetQuery/QueryArray/GetQueryArray/QueryMap/GetQueryMap`，懒加载 `queryCache` | `context.go:535/548/564/573/590/597/604/578` |
| PostForm 取值 | `PostForm/DefaultPostForm/GetPostForm/PostFormArray/...`，`initFormCache` 调 `ParseMultipartForm(MaxMultipartMemory)` | `context.go:611/619/634/643/648/663/670/677` |
| 文件上传 | `FormFile/MultipartForm/SaveUploadedFile` | `context.go:708/723/734` |
| 绑定（Must 系） | `Bind/BindJSON/BindXML/BindQuery/BindYAML/BindTOML/BindPlain/BindHeader/BindUri`，失败即 Abort 400/413 | `context.go:780-828` |
| 绑定（Should 系） | `ShouldBind*/ShouldBindWith`，只返回错误不中断；`ShouldBindBodyWith` 缓存 body 可重复读 | `context.go:861-991` |
| 渲染快捷 | `Render` 统一入口；`HTML/JSON/IndentedJSON/SecureJSON/JSONP/PureJSON/AsciiJSON/XML/YAML/TOML/ProtoBuf/BSON/String/Redirect/Data/DataFromReader/File/FileAttachment` | `context.go:1202-1372` |
| SSE / 流式 | `SSEvent`、`Stream(step)` 轮询 `CloseNotify` 探测客户端断开 | `context.go:1374/1383` |
| 内容协商 | `Negotiate/NegotiateFormat/SetAccepted`，按 `Accept` 头选格式 | `context.go:1419/1455/1486` |
| 客户端 IP | `ClientIP`：TrustedPlatform → AppEngine 兼容 → unix socket 信任 → trusted proxy 校验 → `validateHeader` 反解 X-Forwarded-For | `context.go:998/1050/1078` |
| Cookie / Header | `SetCookie/SetCookieData/Cookie/Header/GetHeader/Status` | `context.go:1152-1192/1123/1130/1139` |
| context.Context 适配 | `Deadline/Done/Err/Value` 委托给 `Request.Context()`，受 `ContextWithFallback` 开关控制；`Value(ContextKey)` 返回自身 | `context.go:1502/1510/1518/1528/1495` |
| 异步安全拷贝 | `Copy()` 深拷贝 Keys/Params/Errors/Accepted，供 goroutine 使用；`index=abortIndex` 防止拷贝后误推进链 | `context.go:122` |
| 错误收集 | `Error(err)` 包装为 `*Error` 追加到 `c.Errors`；nil 会 panic | `context.go:262` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Context` struct | `context.go:61` | 每请求上下文：内嵌 `responseWriter writermem`、`Request`、`Writer ResponseWriter`、`Params`、`handlers HandlersChain`、`index int8`、`Keys map`、`Errors errorMsgs`、`queryCache/formCache`、反向指针 `engine` |
| `abortIndex int8 = math.MaxInt8>>1` | `context.go:57` | 中断哨兵：`index >= abortIndex` 即 `IsAborted` |
| `HandlerFunc` | `gin.go:51` | `func(*Context)`，链上每步签名 |
| `render.Render` 接口 | `render/render.go` | `Render(ResponseWriter) error` + `WriteContentType`；`c.Render` 面向该接口编程（依赖倒置） |
| `binding.Binding` / `binding.BindingBody` | `binding/binding.go` | `Bind(*http.Request, any) error`；`BindingBody` 直接吃 `[]byte` |
| `Error` / `errorMsgs` | `errors.go` | 绑定错误类型 `ErrorTypeBind` 等；`c.Errors` 收集链上所有错误 |
| `Params` / `Param` | `tree.go` | 路径参数切片，`ByName` 取值 |
| `ContextKey` / `ContextRequestKey` / `BodyBytesKey` | `context.go:50/54/47` | `context.Value()` 约定键；body 缓存键 |
| `Negotiate` struct | `context.go:1405` | 内容协商输出表（Offered/HTMLName/JSON/Data...） |

## 3. 关键调用链

### 调用链 A：中间件链推进与中断（`Context.Next`）
1. Engine 命中路由后设置 `c.handlers = value.handlers`（`gin.go:720`），`c.index` 经 `reset()` 为 -1（`context.go:107`）。
2. 进入 `c.Next()`（`context.go:198`）：`c.index++`，当 `index < len(handlers)` 时依次调用 `c.handlers[index](c)`，每步后 `index++`。
3. 中间件内部典型写法：先做前置处理，再调 `c.Next()` 触发下游；返回后做后置处理。这形成「洋葱模型」。
4. 任一中间件调 `c.Abort()`（`context.go:217`）把 `index = abortIndex`；当前 handler 继续执行完，但 `Next()` 外层循环条件 `index < len(handlers)` 立即失败，后续 handler 不再执行。`AbortWithStatus`（`context.go:223`）= `Status(code)` + `WriteHeaderNow()` + `Abort()`。
5. `Render` 失败时（`context.go:1211`）也会 `c.Error(err)` + `c.Abort()`，阻止后续 handler 写响应。

### 调用链 B：客户端 IP 解析（`Context.ClientIP`）
1. 若 `engine.TrustedPlatform` 非空，直接读对应平台头（如 `CF-Connecting-IP`）返回（`context.go:1000-1005`）。
2. 兼容旧 `AppEngine` 标志（已弃用，打印日志）（`context.go:1008-1013`）。
3. unix socket 监听直接信任（`context.go:1020-1023`）；否则 `RemoteIP()`（`context.go:1050`）从 `Request.RemoteAddr` 剥端口，`engine.isTrustedProxy(ip)` 判断是否在 trustedCIDRs 内（`context.go:1034`）。
4. 仅当远端是可信代理且 `ForwardedByClientIP` 开启时，遍历 `RemoteIPHeaders`，调 `engine.validateHeader(header)` 反解 X-Forwarded-For（`context.go:1037-1044`，`gin.go:482`）；否则回退到直连 IP。

### 调用链 C：请求体绑定（`ShouldBindBodyWith` 可重复读）
1. `c.Bind(obj)`（`context.go:780`）按 `Method+ContentType` 选 binding（`binding.Default`），走 `MustBindWith`。
2. `MustBindWith`（`context.go:833`）调 `ShouldBindWith` → `b.Bind(c.Request, obj)`；错误时按 `errors.As` 区分 `*http.MaxBytesError` → 413，其余 → 400，并 `AbortWithError`。
3. `ShouldBindBodyWith`（`context.go:951`）先查 `c.Keys[BodyBytesKey]`，未命中则 `io.ReadAll(c.Request.Body)` 并 `c.Set(BodyBytesKey, body)` 缓存；之后对同一 body 多次绑定不同格式都不再读流。

### 调用链 D：响应渲染（`c.JSON` → `c.Render`）
1. `c.JSON(code, obj)`（`context.go:1260`）构造 `render.JSON{Data: obj}` 调 `c.Render`。
2. `Render`（`context.go:1202`）先 `c.Status(code)`；若状态码不允许 body（如 304/204）只写 ContentType 后 `WriteHeaderNow`；否则 `r.Render(c.Writer)` 写 body，失败则 `c.Error(err)` + `c.Abort()`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `c.engine.MaxMultipartMemory` | 32MB，`ParseMultipartForm` 阈值 | `context.go:652`、`gin.go:26` |
| `c.engine.ForwardedByClientIP` | true，ClientIP 是否查代理头 | `context.go:1037` |
| `c.engine.RemoteIPHeaders` | `["X-Forwarded-For","X-Real-IP"]` | `context.go:1038` |
| `c.engine.TrustedPlatform` | 空；设置后直读平台头 | `context.go:1000` |
| `c.engine.ContextWithFallback` | false；开启后 `Deadline/Done/Err/Value` 委托 `Request.Context()` | `context.go:1496` |
| `c.engine.secureJSONPrefix` | `while(1);` | `context.go:1243` |
| `BodyBytesKey` | `_gin-gonic/gin/bodybyteskey`，body 缓存键 | `context.go:47` |
| `ContextKey` / `ContextRequestKey` | `_gin-gonic/gin/contextkey` / 0 | `context.go:50/54` |
| `bindinding.Validator` | 默认 `go-playground/validator`；`DisableBindValidation()` 关闭 | `mode.go:81` |
| `binding.EnableDecoderUseNumber` / `DisallowUnknownFields` | 全局 JSON decoder 开关 | `mode.go:87/93` |

## 5. 错误与重试语义

- **`c.Error(nil)` panic**（`context.go:263`）：编程错误，启动期/运行期快速失败。
- **`MustGet` 不存在 key panic**（`context.go:310`）：编程错误。
- **绑定错误**：`MustBindWith` 不重试，直接 `AbortWithError(400 或 413)` 中断链；错误类型 `ErrorTypeBind`。`ShouldBind*` 不中断，把 error 返回给 handler 自行处理。
- **渲染错误**：`r.Render` 返回 error 时 `c.Error(err)` + `c.Abort()`（`context.go:1211-1215`），不再写 body。
- **表单解析错误**：`ParseMultipartForm` 非 `ErrNotMultipart` 时 `debugPrint` 记录，不中断（`context.go:653-655`）。
- **`Stream` 客户端断开**：`CloseNotify` 通道关闭即返回 `true`（`context.go:1388`），由 handler 决定收尾。
- 本叶子**无重试队列/退避**；错误要么 panic（编程错），要么转 HTTP 状态码 / 收集到 `c.Errors`。

## 6. 并发细节

- **`Context` 本身单 goroutine 内使用**：每请求由 `net/http` 起独立 goroutine，Context 字段不跨请求共享（除 `pool.Get` 复用）。
- **`c.mu sync.RWMutex`**（`context.go:76`）：仅保护 `c.Keys` map；`Set` 写锁、`Get`/`getTyped` 读锁。`reset()` 把 `c.Keys=nil` 不持锁（仅在 pool Get 后单线程调用）。
- **`Copy()` 用于跨 goroutine**（`context.go:122`）：深拷贝 Keys（`maps.Clone`）、Params、Errors、Accepted；`writermem.ResponseWriter=nil` 禁止拷贝后写响应；`index=abortIndex` 防止误推进链。
- **`queryCache`/`formCache`**：懒加载，首次取值时一次性解析；同请求内复用，避免重复 `url.Query()` / `ParseMultipartForm`。
- **`context.Context` 委托**：`Done()` 在未开 fallback 时返回 `nil`（永不就绪）；`Value` 先查 `ContextRequestKey`/`ContextKey`/字符串键，最后委托 `Request.Context().Value`（`context.go:1528-1543`）。
- **无 channel/workqueue**：`Stream` 用 `select` 监听 `CloseNotify` 通道，但这是响应写循环，不是内部任务调度。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `context.go`：Context 全部方法、链式协议、取值、绑定快捷、渲染快捷、SSE/Stream、内容协商、context.Context 适配
- `errors.go`：`Error`/`errorMsgs` 类型与 `SetType`
- 与 `response_writer.go`（`writermem`）、`gin.go`（engine 反向指针）、`tree.go`（Params）同包

**Out-of-Scope（不在本仓库源码内）**
- `net/http.Request`/`ResponseWriter`：Go 标准库
- `binding/` 子包：JSON/XML/Form/Uri 等具体解码与校验算法（本叶子只做入口与错误转 HTTP 状态）
- `render/` 子包：各格式具体序列化实现（本叶子只做 `Render` 分发）
- `quic-go`/`http2`：传输层
- 中间件具体实现（Logger/Recovery）在 `logger.go`/`recovery.go`，属 middleware 域

## 8. 与相邻子系统交互

- **上游 → engine 叶子**：Engine 从 `pool` 取 Context，填 `c.handlers` 后调 `c.Next()`（`gin.go:722`）。
- **上游 → routing 域**：`root.getValue` 把路径参数写进 `c.params`，Engine 再赋给 `c.Params`（`gin.go:715-718`）。
- **下游 → response-writer 叶子**：`c.Writer` 接口由 `writermem` 实现；`Status/Header/WriteHeaderNow/Flush` 全部转发到 `writermem`。
- **下游 → binding 子包**：`ShouldBindWith` 调 `b.Bind(c.Request, obj)`（`context.go:943`）；`ShouldBindBodyWith` 调 `bb.BindBody(body, obj)`。
- **下游 → render 子包**：所有 `c.JSON/HTML/...` 构造 render 对象后交 `c.Render`；`c.engine.HTMLRender.Instance(name, obj)` 取 HTML 实例（`context.go:1227`）。
- **下游 → engine 叶子**：`ClientIP` 回调 `engine.isTrustedProxy`/`engine.validateHeader`（`context.go:1034/1040`）。

## 9. 语言专项适配口径（Go）

- **并发模型**：每请求一 goroutine；`Context` 不是 K8s 控制器，无 Reconcile/informer。同步原语仅 `sync.RWMutex`（保护 Keys，`context.go:76`）。跨 goroutine 必须用 `Copy()`（`context.go:122`）——这是 gin 对 Go 并发安全的核心约束。
- **context.Context 传递**：`Context` 自身实现 `context.Context` 接口（`Deadline/Done/Err/Value`，`context.go:1502-1543`），可作为 `context.Context` 传给下游；默认（`ContextWithFallback=false`）下 `Done()` 返回 nil，避免旧行为改变。
- **多二进制**：无 cmd/，纯库；Context 由用户 main 通过 Engine 间接使用。
- **internal 边界**：`internal/bytesconv`/`internal/fs` 仅包内可见；`Context` 对外暴露的是 `Writer ResponseWriter` 接口（response_writer.go 定义），依赖倒置——handler 面向接口编程。
- **泛型使用**：`getTyped[T any]`（`context.go:313`）消除 `GetString/GetInt/...` 几十个重复样板，是 Go 1.18+ 泛型在 gin 内的典型应用。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| Context 结构图 | `context-architecture.html` | architecture | showcase（一次通过） |
| 中间件链时序图 | `context-sequence.html` | sequence | showcase（首轮自调用边跨距 0、y 越界，改纯交互消息后通过） |
| 请求绑定/渲染数据流图 | `context-dataflow.html` | dataflow | showcase（首轮 label 重叠、viewBox 不足，分流路由+labelDy 后通过） |

- 本叶子未生成 workflow/lifecycle 图：链式执行的"流程"语义已由 sequence 图表达；Context 无独立实体状态机（`index` 是游标不是业务状态）。
- JSON IR 位于 `json/` 目录。
