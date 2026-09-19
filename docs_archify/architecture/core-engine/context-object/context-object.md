# 上下文对象（context-object）

> 本文是 `core-engine` 域下的叶子子系统文档。域级总览见 `../core-engine.md`。
> 本文只展开 **Context 的结构、生命周期复用、handler 链流控、取值/绑定/渲染方法族与 context.Context 适配**，不重复展开 Engine 如何取放 Context（见 `../engine-lifecycle/engine-lifecycle.md`）、路由参数如何被填进 `Params`（见 `../../router-tree/radix-tree/radix-tree.md`）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。主源码文件：`context.go`（约 144 个方法）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Context 结构体 | 聚合 writermem/Request/Writer/Params/handlers/Keys/Errors/缓存等请求态 | `context.go:61` |
| 请求态重置 | `reset()` 把切片长度归零、指针字段清空，供 sync.Pool 复用 | `context.go:103` |
| 后台安全拷贝 | `Copy()` 深拷 Keys/Params/Errors/Accepted，供传入后台 goroutine | `context.go:122` |
| handler 链流控 | `Next()` 自增 index 顺序执行；`Abort()` 置 `abortIndex` 阻断；`IsAborted()` | `context.go:198`、`context.go:217`、`context.go:209` |
| Abort 便捷族 | `AbortWithStatus/AbortWithStatusJSON/AbortWithStatusPureJSON/AbortWithError` | `context.go:223`、`context.go:240`、`context.go:232`、`context.go:248` |
| 错误收集 | `Error(err)` 包装为 `*Error` 追加到 `Errors`（nil 会 panic） | `context.go:262` |
| 请求级 KV | `Set/Get/MustGet/Delete` + 30+ 类型化 getter（GetString/GetInt...） | `context.go:286`、`context.go:298`、`context.go:306`、`context.go:492` |
| 路由参数取值 | `Param`/`AddParam`（基于 `Params.ByName`） | `context.go:513`、`context.go:522` |
| Query 取值族 | `Query/GetQuery/QueryArray/QueryMap/DefaultQuery`，懒缓存 `initQueryCache` | `context.go:535`、`context.go:564`、`context.go:573`、`context.go:597`、`context.go:548`、`context.go:578` |
| Form 取值族 | `PostForm/GetPostForm/PostFormArray/PostFormMap`，懒缓存 `initFormCache` | `context.go:611`、`context.go:634`、`context.go:643`、`context.go:670`、`context.go:648` |
| 文件上传 | `FormFile/MultipartForm/SaveUploadedFile` | `context.go:708`、`context.go:723`、`context.go:734` |
| 绑定族 | `Bind*`（4xx 中止）vs `ShouldBind*`（不中止）；`MustBindWith/ShouldBindWith/ShouldBindBodyWith` | `context.go:780`、`context.go:833`、`context.go:861`、`context.go:942`、`context.go:951` |
| 渲染族 | `Render/JSON/IndentedJSON/SecureJSON/JSONP/AsciiJSON/PureJSON/XML/YAML/TOML/ProtoBuf/BSON/String/Data/HTML/Redirect` | `context.go:1202`、`context.go:1260`、`context.go:1221`、`context.go:1314` |
| 静态文件 | `File/FileFromFS/FileAttachment` | `context.go:1341`、`context.go:1346`、`context.go:1364` |
| SSE / 流 | `SSEvent/Stream` | `context.go:1374`、`context.go:1383` |
| 内容协商 | `Negotiate/NegotiateFormat/SetAccepted` | `context.go:1419`、`context.go:1455`、`context.go:1486` |
| 客户端 IP | `ClientIP/RemoteIP`（依赖 Engine 可信代理） | `context.go:998`、`context.go:1050` |
| Cookie / Header | `SetCookie/GetCookie/SetSameSite/Status/Header/GetHeader` | `context.go:1159`、`context.go:1192`、`context.go:1123`、`context.go:1130` |
| context.Context 适配 | `Deadline/Done/Err/Value`（受 `ContextWithFallback` 控制） | `context.go:1502`、`context.go:1510`、`context.go:1518`、`context.go:1528` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `Context` struct | `context.go:61` | 请求级上下文，同时充当 handler 入参与 `context.Context` |
| `abortIndex int8 = math.MaxInt8>>1` | `context.go:57` | 中止哨兵值；`Abort()` 把 `index` 置此值使 `Next()` 循环立即退出 |
| `ContextKeyType int` / `ContextRequestKey` | `context.go:52` / `context.go:54` | 私有 key，`Value()` 据此返回 `c.Request` |
| `ContextKey` / `BodyBytesKey` 常量 | `context.go:50` / `context.go:47` | `Value()` 返回 Context 自身的 key；body 复用缓存的 key |
| `errorMsgs` / `Error`（见 errors.go） | — | `c.Errors` 收集错误的类型 |
| `HandlersChain`（见 gin.go） | `gin.go:57` | `c.handlers` 持有的中间件+业务链 |
| `ResponseWriter` 接口 | `response_writer.go:23` | `c.Writer` 的类型，组合 `http.ResponseWriter/Hijacker/Flusher/CloseNotifier` |

## 3. 关键调用链

**链路 A：handler 链顺序执行（中间件洋葱模型）**

1. 路由命中后 Engine 把 `value.handlers` 赋给 `c.handlers`，调 `c.Next()`（`gin.go:722` → `context.go:198`）。
2. `Next()` 先 `c.index++`，再循环 `for c.index < safeInt8(len(c.handlers))` 依次调用 `c.handlers[c.index](c)` 并 `c.index++`（`context.go:199-205`）。
3. 中间件在自身逻辑里再次调 `c.Next()`，从而嵌套进入下一个 handler，形成"洋葱"——前半段在 `Next()` 之前执行，后半段在 `Next()` 返回后执行。
4. 任一中间件调 `c.Abort()` 把 `c.index = abortIndex`（`context.go:218`），`IsAborted()` 即 `c.index >= abortIndex`（`context.go:210`），后续 `Next()` 循环条件立即不成立。

**链路 B：绑定（以 MustBind 为例）**

1. `c.Bind(obj)` 用 `binding.Default(method, contentType)` 选绑定器，再 `MustBindWith(obj, b)`（`context.go:781-782`）。
2. `MustBindWith` 先 `ShouldBindWith`（即 `b.Bind(c.Request, obj)`，`context.go:943`）；出错时按 `errors.As(err, &maxBytesErr)` 区分 `413` 与 `400`，并 `AbortWithError(...).SetType(ErrorTypeBind)`（`context.go:841-846`）。
3. `ShouldBindBodyWith` 为支持重复绑定，先从 `c.Get(BodyBytesKey)` 取已读 body，未命中则 `io.ReadAll(c.Request.Body)` 读一次并 `c.Set(BodyBytesKey, body)` 缓存（`context.go:953-963`）。

**链路 C：取值带懒缓存**

- `GetQueryArray` 先 `initQueryCache()`（`context.go:591`）：首次才 `c.Request.URL.Query()` 并缓存到 `c.queryCache`（`context.go:578-586`）。
- `GetPostFormArray` 先 `initFormCache()`（`context.go:664`）：首次 `ParseMultipartForm(MaxMultipartMemory)` 后把 `req.PostForm` 赋给 `c.formCache`（`context.go:648-658`）。

**链路 D：作为 context.Context 使用**

- `Value(key)`：命中 `ContextRequestKey` 返回 `c.Request`，命中 `ContextKey` 返回 `c`，string key 先查 `c.Get`，否则在 `hasRequestContext()`（`ContextWithFallback && Request.Context()!=nil`）为真时透传给 `c.Request.Context().Value(key)`（`context.go:1528-1543`）。

## 4. 配置项

| 配置项 / option | 默认 / 行为 | 位置 |
|------|------|------|
| `engine.MaxMultipartMemory` | 32MB，`initFormCache` 传给 `ParseMultipartForm` | `context.go:652` |
| `engine.ContextWithFallback` | 默认 `false`；开启后 Context 才对外当 `context.Context` 回退 | `context.go:1496` |
| `engine.TrustedPlatform/ForwardedByClientIP/RemoteIPHeaders` | 影响 `ClientIP()` 解析 | `context.go:1000`、`context.go:1037` |
| `c.sameSite` | 由 `SetSameSite` 设置，影响后续 `SetCookie` | `context.go:1152`、`context.go:96` |
| `c.Accepted` | `SetAccepted` 手动覆盖内容协商候选格式 | `context.go:1486` |

## 5. 错误与重试语义

- **绑定错误**：`Bind*` 系列自动 `AbortWithError`（400/413）并中断链；`ShouldBind*` 系列只返回 error，由调用方决定如何响应，不自动 Abort。
- **nil error panic**：`Error(nil)` 直接 panic（`context.go:263-265`），约定调用方不得传 nil。
- **MaxBytes 区分**：`MustBindWith` 用 `errors.As` 区分 `http.MaxBytesError`（413）与普通绑定错误（400），注释说明 sonic/go-json 不一定传播该错误（`context.go:836-843`）。
- **解析 form 错误**：`initFormCache` 对 `ParseMultipartForm` 出错时，忽略 `http.ErrNotMultipart` 仅 `debugPrint` 其它错误（`context.go:653-655`）。
- **无重试**：Context 自身不做任何重试；重试/退避由业务或中间件实现。

## 6. 并发细节

- **goroutine 边界**：默认 Context 绑定在单个请求 goroutine 内，**禁止**直接传给后台 goroutine；必须先 `c.Copy()`（`context.go:122`）。`Copy()` 把 `index` 置 `abortIndex`、`handlers=nil`、`ResponseWriter=nil`，切断与原请求的链与写。
- **Keys 锁**：`c.mu sync.RWMutex` 保护 `Keys map[any]any`；`Set` 写锁（`context.go:287`），`Copy` 读锁后 `maps.Clone`（`context.go:136-138`）。其它字段（Params/queryCache/formCache/handlers）单请求内不并发访问，故无锁。
- **Pool 复用与零分配**：`reset()`（`context.go:103`）把 `Params`/`Errors`/`*c.params`/`*c.skippedNodes` 长度归零但保留底层数组，配合 `Engine.pool` 实现复用（见 `engine-lifecycle`）。
- **context.Context 传播**：`Done/Err/Deadline` 在开启 fallback 时透传 `c.Request.Context()`，请求取消沿 handler 链与下游 RPC 传播；`Value` 是 gin 自定义 key 与标准 request ctx 的桥接点。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `context.go` 全文：Context 的全部方法族。
- 与 `binding/`（绑定器）、`render/`（渲染器）、`response_writer.go`（Writer 包装）的调用契约。

**Out-of-Scope（不在本仓库源码内）**
- `binding/` 内部各格式解析与 `go-playground/validator` 校验实现（外部依赖）。
- `render/` 各 Render 的字节序列化（JSON 后端 sonic/jsoniter/go-json 为外部依赖）。
- `net/http` 的 Request/ResponseWriter 本身。

**不做什么**：不负责从 sync.Pool 取放 Context（Engine 负责）、不负责把路由树结果填入 `Params`（radix-tree 负责）、不实现绑定器/渲染器本体。

## 8. 与相邻子系统交互

- **Engine → Context**：`ServeHTTP` 取 Context → `reset()` → 路由命中后填 `handlers/Params` → `c.Next()` → 归还。
- **radix-tree → Context**：`getValue` 把 `:param/*` 写入 `c.params`，命中后 Engine 赋给 `c.handlers`。
- **Context → response_writer**：所有渲染/写响应最终走 `c.Writer`（即 `c.writermem`），`WriteHeaderNow` 落头。
- **Context → binding/**：`Bind*/ShouldBind*` 调 `b.Bind`/`b.BindBody`。
- **Context → render/**：`Render/JSON/HTML` 调 `r.Render(c.Writer)`/`WriteContentType`。
- **Context → Engine（反向）**：`ClientIP()` 调 `engine.isTrustedProxy/validateHeader`。

## 9. 语言专项适配口径

- **并发模型**：典型单请求 goroutine 内对象 + 显式 `Copy()` 逃逸到后台；`sync.RWMutex` 仅保护 `Keys` map，体现"最小加锁"——热点路径（取值/绑定）不加锁，只在跨 goroutine 共享点加锁。
- **context.Context 桥接**：gin 的 `Context` 同时实现 `context.Context` 接口（`Value/Deadline/Done/Err`），是把 gin 自定义 KV 接入标准 `context.Context` 取消/超时体系的关键适配层；`ContextWithFallback` 作为开关避免无谓透传。
- **sync.Pool 零分配**：`reset()` 保容量复用 + `allocateContext` 预分配 maxParams/maxSections 容量，是 gin 高性能路由的核心（与 `radix-tree` 零分配设计呼应）。
- **internal 边界**：本叶子不直接引用 internal 包；`maps.Clone` 用标准库。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Context 组成与方法族架构图 | `context-object-architecture.html` | architecture | standard（降档原因：左右对称 6 个外部/相邻节点的边标签间距在 showcase 严格校验下反复触发重叠，按 usage-guide 第 4.6 节降 standard，render 退出码 0） |

- JSON IR 源文件位于 `json/context-object-architecture.json`。
- 本叶子**不补时序图**：理由——handler 链洋葱模型的时序已在 `engine-lifecycle-sequence.html` 主路径中体现（`c.Next()` 一段），本叶子用第 3 节文字化补充 Abort/嵌套语义，避免重复。
