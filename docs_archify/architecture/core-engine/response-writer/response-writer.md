# 响应写出器（response-writer）

> 本文是 `core-engine` 域下的叶子子系统文档。域级总览见 `../core-engine.md`。
> 本文只展开 **gin 对 `http.ResponseWriter` 的拦截包装（捕获 Status/Size、延迟落头）以及静态文件系统的目录列表关闭包装**，不重复展开 Context 如何调用 Writer（见 `../context-object/context-object.md`）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。主源码文件：`response_writer.go`、`fs.go`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| ResponseWriter 接口 | 组合 `http.ResponseWriter/Hijacker/Flusher/CloseNotifier` + gin 自增方法 | `response_writer.go:23` |
| responseWriter 实现 | 内嵌底层 `http.ResponseWriter`，叠加 `size/status` 两个状态字段 | `response_writer.go:49` |
| 状态/字节捕获 | `Status()`/`Size()`/`Written()` 暴露当前状态码与已写字节数 | `response_writer.go:98`、`response_writer.go:102`、`response_writer.go:106` |
| 延迟写头 | `WriteHeader(code)` 只记录状态码；`WriteHeaderNow()` 真正下发头 | `response_writer.go:67`、`response_writer.go:77` |
| 写body | `Write`/`WriteString` 先 `WriteHeaderNow` 再透传底层，累加 size | `response_writer.go:84`、`response_writer.go:91` |
| 重写状态码保护 | 已写过后再 `WriteHeader` 打警告且忽略，不覆盖 | `response_writer.go:69-72` |
| Hijack | 实现 `http.Hijacker`：body 未写（size≤0）才允许劫持，否则报错；websocket 兼容 | `response_writer.go:111` |
| Flush / CloseNotify / Pusher | 类型断言透传底层 `http.Flusher/CloseNotifier/Pusher` | `response_writer.go:136`、`response_writer.go:128`、`response_writer.go:143` |
| Unwrap | 支持 `http.ResponseController` 等用 `Unwrap` 取底层 writer | `response_writer.go:57` |
| 状态重置 | `reset(writer)`：换底层 writer、size=`noWritten(-1)`、status=`200` | `response_writer.go:61` |
| 关闭目录列表 | `OnlyFilesFS` 包装 `http.FileSystem`，`Readdir` 恒返回 nil | `fs.go:13`、`fs.go:33` |
| 目录工厂 | `Dir(root, listDirectory)`：true 返回 `http.Dir`，false 包成 `OnlyFilesFS` | `fs.go:42` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `ResponseWriter` interface | `response_writer.go:23` | gin 对外暴露的 Writer 类型（`c.Writer` 的静态类型） |
| `responseWriter` struct | `response_writer.go:49` | 私有实现，`var _ ResponseWriter = (*responseWriter)(nil)` 在 `response_writer.go:55` |
| `noWritten=-1` / `defaultStatus=200` | `response_writer.go:16-17` | size 哨兵与默认状态码 |
| `errHijackAlreadyWritten` | `response_writer.go:20` | body 已写后劫持的错误 |
| `OnlyFilesFS` struct | `fs.go:13` | 关闭 `Readdir` 的文件系统包装 |
| `neutralizedReaddirFile` struct | `fs.go:28` | 包住单个 `http.File`，重写 `Readdir` 返回 nil |

## 3. 关键调用链

**链路 A：延迟落头与写 body**

1. 请求开始时 Engine 调 `c.writermem.reset(w)`（`response_writer.go:61`）：`size=noWritten(-1)`、`status=200`。
2. handler 调 `c.JSON(200, obj)` 等最终走 `Write(data)`（`response_writer.go:84`）：先 `WriteHeaderNow()`——若 `!Written()`（size 仍为 -1）则 `size=0` 并 `w.ResponseWriter.WriteHeader(w.status)` 真正下发状态码与头。
3. 随后 `w.ResponseWriter.Write(data)` 透传底层，`w.size += n` 累加字节数（`response_writer.go:86-87`）。
4. 中间件可用 `c.Writer.Status()`/`Size()` 在响应完成后读到真实状态码与字节数（Logger 中间件据此计时与记录）。

**链路 B：状态码设置与重写保护**

- `WriteHeader(code)`：若 `code>0` 且与当前不同，且 `Written()` 已真（body 已写），只 `debugPrint` 警告并 return，不覆盖（`response_writer.go:68-72`）；否则记录 `w.status=code`。
- 这意味着业务中途 `c.Status(404)` 只改状态码，直到 `WriteHeaderNow` 才落头，中间件可在 `Next()` 之后读到。

**链路 C：Hijack（websocket 场景）**

1. 调 `Hijack()`：若 `w.size > 0`（body 已写字节）返回 `errHijackAlreadyWritten`（`response_writer.go:114-116`）。
2. 断言底层是否实现 `http.Hijacker`，不支持返回 `http.ErrNotSupported`（`response_writer.go:117-120`）。
3. 劫持前若 `size<0`（还没写过）先置 `size=0`，再调底层 `Hijack()`（`response_writer.go:121-124`）。

**链路 D：静态文件关闭目录列表**

- `Dir(root, false)` 返回 `&OnlyFilesFS{FileSystem: http.Dir(root)}`（`fs.go:49`）；`Open` 每个文件都包成 `neutralizedReaddirFile`（`fs.go:18-25`），其 `Readdir` 恒 `return nil, nil`（`fs.go:33-36`），从而 `http.FileServer` 列出目录时返回空、不暴露文件清单。

## 4. 配置项

| 配置项 / 行为 | 默认 / 说明 | 位置 |
|------|------|------|
| 默认状态码 | `defaultStatus = http.StatusOK(200)`，未显式 `c.Status` 时按 200 落头 | `response_writer.go:17` |
| 未写哨兵 | `noWritten = -1`，`Written()` 即 `size != -1` | `response_writer.go:16`、`response_writer.go:107` |
| `Dir(root, listDirectory)` | listDirectory=false 才包 OnlyFilesFS；true 直接返回 `http.Dir` | `fs.go:45-49` |
| Hijack 阈值 | body 已写（size>0）即拒绝劫持 | `response_writer.go:114` |

## 5. 错误与重试语义

- **重写状态码**：已写 body 后再 `WriteHeader` 不报错只警告，避免破坏已下发响应。
- **Hijack 错误**：返回 `errHijackAlreadyWritten` 或 `http.ErrNotSupported`，调用方（websocket 库）自行处理。
- **Flush/Pusher 不支持**：类型断言失败时静默不调（`Flush` 仅 `if ok` 透传），不 panic。
- **无重试**：Writer 是同步透传，不做任何重试。

## 6. 并发细节

- **单请求单 writer**：`responseWriter` 绑定在单个请求 Context 上，由该请求 goroutine 串行写，无并发写冲突。
- **池化重置**：`reset(writer)` 由 `Engine.ServeHTTP` 在 `pool.Get` 后调用（见 engine-lifecycle），每次请求换新底层 `http.ResponseWriter` 并复位 size/status。
- **CloseNotify channel**：`CloseNotify()` 透传底层返回的 `<-chan bool`，不额外加锁。
- **无 mutex**：状态字段 size/status 单 goroutine 访问，无需锁；这是高性能捕获设计。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `response_writer.go` 全文：Writer 拦截与状态捕获。
- `fs.go`：目录列表关闭包装。

**Out-of-Scope（不在本仓库源码内）**
- 底层 `http.ResponseWriter`/`http.Hijacker`/`http.Flusher`/`http.Pusher`（Go 标准库 `net/http`）。
- `http.FileSystem`/`http.File`/`http.Dir`（Go 标准库 `net/http`）。
- websocket 库（如 gorilla/coder）对 Hijack 的消费方。

**不做什么**：不决定写什么内容（Context 渲染族负责）、不做缓冲池（sync.Pool 在 Engine）、不实现 http.FileServer（标准库）。

## 8. 与相邻子系统交互

- **Engine → responseWriter**：`ServeHTTP` 里 `c.writermem.reset(w)` 把底层标准库 writer 包进来。
- **Context → responseWriter**：所有 `Render/JSON/HTML/Redirect` 最终调 `c.Writer.Write/WriteHeader`；中间件读 `Status()/Size()`。
- **responseWriter → 标准库**：`Write/WriteHeader/Hijack/Flush/Pusher` 全部透传底层 `http.ResponseWriter`。
- **router-group → fs**：`Static` 用 `Dir(root, false)` 得到 `OnlyFilesFS`，交给 `http.FileServer`。
- **标准库 FileServer → OnlyFilesFS**：列目录时 `Readdir` 返回 nil，关闭目录暴露。

## 9. 语言专项适配口径

- **接口组合与类型断言**：`ResponseWriter` 内嵌 `http.ResponseWriter` 并组合 Hijacker/Flusher/CloseNotifier，gin 通过接口断言把"可选能力"（Hijack/Flush/Push）做成 best-effort 透传——这是 Go 标准库可选接口模式的典型用法。
- **Unwrap 支持**：`Unwrap()` 让 `http.ResponseController` 等 Go 1.20+ 新设施能穿过包装直接访问底层 writer（Go 标准库约定的包装协议）。
- **延迟落头（deferred WriteHeader）**：`status` 先记录、`WriteHeaderNow` 才下发，使 gin 能在中间件链执行完后据真实分支落头——这是对标准库"Write 即隐式 200"的精细化拦截。
- **无锁单 goroutine**：与 Context 零分配设计一脉相承，状态字段不加锁。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 响应写出器拦截架构图 | `response-writer-architecture.html` | architecture | showcase |

- JSON IR 源文件位于 `json/response-writer-architecture.json`。
- 本叶子**不补时序图**：理由——`reset → WriteHeader(记录) → Write(透传+累加)` 的写主路径在 `engine-lifecycle-sequence.html` 的 `c.Next() → 写响应` 中已覆盖，本叶子用第 3 节文字化补充延迟落头语义。
