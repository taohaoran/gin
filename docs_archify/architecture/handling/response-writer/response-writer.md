# 响应写入器（response-writer）

> 本文是 `handling` 域下的叶子子系统文档。域级总览见 `../handling.md`。
> 本文只展开 **`ResponseWriter` 接口与 `responseWriter` 对 `net/http.ResponseWriter` 的包装**（status/size 跟踪、延迟写头、Hijack/Flush/CloseNotify/Pusher 透传），不重复展开 Context 的渲染方法（见 context 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 接口扩展 | `ResponseWriter` 接口组合 `http.ResponseWriter` + `Hijacker` + `Flusher` + `CloseNotifier`，并扩展 `Status()/Size()/Written()/WriteString()/WriteHeaderNow()/Pusher()` | `response_writer.go:23-47` |
| 状态包装 | `responseWriter` struct 内嵌底层 `http.ResponseWriter`，额外记录 `status int` 与 `size int` | `response_writer.go:49-53` |
| 复用 reset | 每请求 `reset(w)` 绑底层 writer、`size=noWritten(-1)`、`status=200` | `response_writer.go:61-65` |
| 延迟写头 | `WriteHeader(code)` 只记录 status（不立即下发），重复写只告警不覆盖 | `response_writer.go:67-75` |
| 强制落头 | `WriteHeaderNow()` 未写则 `size=0` 并下发 `WriteHeader(status)` | `response_writer.go:77-82` |
| 写 body | `Write/WriteString` 先 `WriteHeaderNow()` 再写底层，累计 size | `response_writer.go:84-96` |
| 状态自省 | `Status()/Size()/Written()`（size != -1 即已写） | `response_writer.go:98-108` |
| Hijack | 支持 WebSocket 升级：size<0 或 ==0 可劫持，size>0 返回 `errHijackAlreadyWritten` | `response_writer.go:111-125` |
| CloseNotify | 透传底层 `http.CloseNotifier`（客户端断连通知） | `response_writer.go:128-133` |
| Flush | 先落头再透传 `http.Flusher`（SSE/分块） | `response_writer.go:136-141` |
| Pusher | 透传 `http.Pusher`（HTTP/2 server push） | `response_writer.go:143-148` |
| Unwrap | 供 `errors.As`/`http.NewResponseController` 等解包底层 writer | `response_writer.go:57-59` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ResponseWriter` interface | `response_writer.go:23` | gin 对外暴露的响应写入接口，组合 4 个标准库接口 + 6 个扩展方法；`Context.Writer` 字段就是此类型 |
| `responseWriter` struct | `response_writer.go:49` | 唯一实现；内嵌 `http.ResponseWriter`，`size`/`status` 两个字段 |
| `noWritten = -1` | `response_writer.go:16` | 「尚未写任何字节」哨兵值 |
| `defaultStatus = 200` | `response_writer.go:17` | 默认状态码 |
| `errHijackAlreadyWritten` | `response_writer.go:20` | body 已写后再 Hijack 的错误 |
| `var _ ResponseWriter = (*responseWriter)(nil)` | `response_writer.go:55` | 编译期接口断言 |

## 3. 关键调用链

### 调用链 A：写响应（延迟落头）
1. Engine 从 pool 取 Context 后调 `c.writermem.reset(w)`（`gin.go:668`、`response_writer.go:61`）：`size=-1, status=200`。
2. handler 调 `c.Status(404)`（`context.go:1123`）→ `w.WriteHeader(404)`（`response_writer.go:67`）：仅记录 `w.status=404`，**不下发**。
3. handler 调 `c.JSON(200, obj)` → `render.JSON.Render` → `w.Write(data)`（`response_writer.go:84`）：先 `WriteHeaderNow()`（`response_writer.go:77`）——发现 `size==-1`，置 `size=0` 并调底层 `WriteHeader(status)`，再 `Write(data)`，`size += n`。
4. Engine 在 `c.Next()` 返回后再兜底调一次 `c.writermem.WriteHeaderNow()`（`gin.go:723`），确保即使 handler 没写 body 也把 status 发出去。

### 调用链 B：Hijack（WebSocket 升级）
1. WebSocket 库调 `c.Writer.(http.Hijacker).Hijack()`（`response_writer.go:111`）。
2. 若 `w.size > 0`（body 已写）直接返回 `errHijackAlreadyWritten`（`response_writer.go:114`）。
3. 若 `w.size < 0`（还没写头）先置 `size=0`，再类型断言底层实现 `http.Hijacker` 并转发（`response_writer.go:117-124`）。

### 调用链 C：Flush（SSE/流式）
1. `c.Stream(step)`（`context.go:1383`）每轮调 `w.Flush()`（`response_writer.go:136`）。
2. `Flush` 先 `WriteHeaderNow()` 落头，再类型断言底层 `http.Flusher` 转发（`response_writer.go:138-140`）；底层不支持 Flush 则静默跳过。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| 默认 status | `200 OK`（`defaultStatus`） | `response_writer.go:17` |
| 未写哨兵 | `size = -1`（`noWritten`） | `response_writer.go:16` |
| 重复 WriteHeader | 仅 debug 告警，不覆盖已设 status | `response_writer.go:69-71` |
| Hijack 时机 | 仅 size ≤ 0 允许；size > 0 报错 | `response_writer.go:114` |
| Flush/Pusher/CloseNotify | 底层不支持时静默降级（返回 nil/空） | `response_writer.go:129/138/144` |

## 5. 错误与重试语义

- **重复写 status**：`WriteHeader` 在已 Written 后再调只 `debugPrint` 告警，不返回错误也不覆盖（`response_writer.go:69-71`）。
- **Hijack 时 body 已写**：返回 `errHijackAlreadyWritten`（`response_writer.go:20/115`），WebSocket 库据此报错；不自动重试。
- **底层不支持 Hijack/Flush/Pusher/CloseNotify**：类型断言失败时返回 `http.ErrNotSupported` 或 nil（`response_writer.go:119/132/146`），由调用方判断。
- **Write 错误**：直接透传底层 `io.Writer.Write` 的 error，gin 不重试（连接已断，重试无意义）。
- 本叶子**无重试/退避**；错误要么透传，要么 debug 告警。

## 6. 并发细节

- **无锁设计**：`responseWriter` 的 `size`/`status` 字段**不加 mutex**——因为每请求独立一个 `responseWriter` 实例（内嵌在 `Context` 里，`Context` 又由 pool 复用），同一请求内串行调用，无并发写。
- **goroutine 边界**：`CloseNotify()` 返回的 `<-chan bool` 由底层 `http.Server` 在客户端断连时关闭；`c.Stream` 在 handler goroutine 内 `select` 该通道（`context.go:1387`）。
- **pool 复用**：`responseWriter` 作为 `Context` 内嵌字段随 Context 一起 `sync.Pool` 复用；`reset()`（`response_writer.go:61`）把 size/status 归零。
- **无 channel/workqueue**；无共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `response_writer.go`：`ResponseWriter` 接口与 `responseWriter` 实现、reset、写头/写 body、Hijack/Flush/CloseNotify/Pusher 透传

**Out-of-Scope（不在本仓库源码内）**
- `net/http.ResponseWriter`/`Hijacker`/`Flusher`/`CloseNotifier`/`Pusher`：Go 标准库接口与底层实现（由 `net/http.Server` 提供）
- WebSocket 库（如 `gorilla/websocket`、`coder/websocket`）：第三方，仅通过 Hijack 接口对接
- 具体渲染逻辑（JSON/HTML 序列化）在 `render/` 子包，本叶子只负责写字节

## 8. 与相邻子系统交互

- **上游 → engine 叶子**：Engine 在 `ServeHTTP` 里 `c.writermem.reset(w)` 绑定标准库 writer（`gin.go:668`）。
- **上游 → context 叶子**：`Context.Writer` 字段类型是 `ResponseWriter`；`c.Status/Header/JSON/HTML/Stream` 等全部转发到 `writermem`。
- **下游 → 标准库**：所有写操作最终委托给内嵌的 `http.ResponseWriter`；Hijack/Flush/Pusher/CloseNotify 通过类型断言动态探测底层能力。
- **下游 → render 子包**：`render.Render(w)` 接收 `ResponseWriter` 接口写字节。

## 9. 语言专项适配口径（Go）

- **并发模型**：无锁、无 goroutine、无 channel；纯粹的「每请求一实例 + 串行调用」。`CloseNotify` 通道是标准库提供的，gin 只透传。
- **接口组合**：`ResponseWriter` 用 Go 接口嵌入组合 4 个标准库接口（`response_writer.go:24-27`），是 Go 接口嵌入的典型用法。
- **类型断言能力探测**：Hijack/Flush/Pusher/CloseNotify 都用 `if x, ok := w.(http.X); ok` 动态探测底层能力，不支持则降级——这是 Go 对可选接口的惯用模式。
- **Unwrap**：实现 `Unwrap() http.ResponseWriter`（`response_writer.go:57`）支持 `http.NewResponseController` 与 errors 解包。
- **多二进制/internal**：纯库，无 cmd/；本文件在根包，不涉及 internal 边界。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| ResponseWriter 架构图 | `response-writer-architecture.html` | architecture | showcase（一次通过） |
| 写响应状态机 | `response-writer-lifecycle.html` | lifecycle | showcase（首轮 state type=initial 非法、缺 main 泳道、标签过短，修正后通过） |

- 本叶子未生成 sequence/dataflow/workflow 图：写响应的时序已由 engine/context 叶子的 sequence 覆盖；无数据管道；无多泳道流程。状态机（size 三态）语义明确，用 lifecycle 表达。
- JSON IR 位于 `json/` 目录。
