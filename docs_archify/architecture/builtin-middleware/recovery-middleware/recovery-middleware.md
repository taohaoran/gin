# Recovery 中间件（recovery-middleware）

> 本文是 `builtin-middleware` 域下的叶子子系统文档。域级总览见 `../builtin-middleware.md`，
> 本文只展开 panic 恢复中间件，不重复日志/认证中间件（见各自叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `recovery.go`（205 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Recovery()` | 默认输出到 `DefaultErrorWriter` 的 panic 恢复中间件 | `recovery.go:35` |
| `CustomRecovery(handle)` | 恢复后调用自定义 `RecoveryFunc` | `recovery.go:40` |
| `RecoveryWithWriter(out, rec...)` | 指定错误输出 Writer，可带恢复回调 | `recovery.go:45` |
| `CustomRecoveryWithWriter(out, handle)` | 核心实现：`defer recover()` 包裹 `c.Next()` | `recovery.go:53` |
| broken pipe 识别 | `EPIPE/ECONNRESET/http.ErrAbortHandler` 不当作需打栈的 panic | `recovery.go:66-68` |
| `secureRequestDump` | 脱敏请求转储，屏蔽 `Authorization` 头 | `recovery.go:98` |
| `defaultHandleRecovery` | 追加错误并 `AbortWithStatus(500)` | `recovery.go:109` |
| `stack(skip)` 栈打印 | `runtime.Caller` 逐帧展开 + 读源码行 | `recovery.go:119` |
| `readNthLine` / `function` | 读指定文件行、裁剪函数名包路径 | `recovery.go:151`、`recovery.go:177` |

对外暴露点：`Recovery()` 是 `gin.Default()` 默认挂载的第二个中间件（`gin.go:239`）。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `RecoveryFunc` | `recovery.go:32` | `func(c *Context, err any)`，用户注入的恢复回调签名 |
| `defaultHandleRecovery` | `recovery.go:109` | 默认恢复动作：包装 error、追加、返回 500 |
| `recoveryWithHandle` 闭包 | `recovery.go:58` | `defer func(){ recover() }()` 包裹 `c.Next()` 的实际中间件 |
| `defaultErrorWriter` 模式 | `recovery.go:56` | `log.New(out, "\n\n\x1b[31m", log.LstdFlags)` 带红色前缀 |

扩展点：`CustomRecoveryWithWriter` 的 `handle RecoveryFunc` 是唯一用户扩展缝——可在 panic 后做上报/告警/自定义状态码。

## 3. 关键调用链

**主路径：下游 panic 被捕获并恢复（`CustomRecoveryWithWriter` 闭包）**

1. 进入中间件后先注册 `defer func(){ if rec := recover(); rec != nil {...} }()`，再调用 `c.Next()`（`recovery.go:59-90`、`recovery.go:90`）。
2. 下游 handler 链中任意位置 panic，Go 运行时沿栈展开，本闭包的 `recover()` 拿到 `rec`（`recovery.go:60`）。
3. 若 `rec` 可断言为 error，用 `errors.Is` 判断是否 broken pipe：`EPIPE`/`ECONNRESET`/`http.ErrAbortHandler`（`recovery.go:64-68`）。
4. 写日志：broken pipe 只打脱敏请求；`IsDebugging()` 为真则额外打完整请求 dump + 栈；否则只打 `[Recovery] ... panic recovered` + 栈（`recovery.go:70-79`）。
5. broken pipe 走 `c.Error(err); c.Abort()`（连接已死不写状态码，`recovery.go:83-84`）；否则调用 `handle(c, rec)`（`recovery.go:86`）。
6. 默认 `defaultHandleRecovery`：把 `rec` 规整为 error，`c.Error(e)` 追加到 `c.Errors`，再 `c.AbortWithStatus(http.StatusInternalServerError)` 返回 500（`recovery.go:114-115`；`AbortWithStatus` 实现在 `context.go:223`，`c.Error` 在 `context.go:262`）。

**栈展开子流程（`stack(stackSkip)`，`recovery.go:119`）**

- `stackSkip=3` 跳过 recover 自身等三帧（`recovery.go:28`）；循环 `runtime.Caller(i)`（`recovery.go:129`），每帧输出 `file:line (pc)`。
- 切换文件时用 `readNthLine` 打开源码文件读第 `line-1` 行（`recovery.go:136`、`recovery.go:156-160`），`function(pc)` 用 `runtime.FuncForPC` 取函数名并裁剪包路径前缀（`recovery.go:177-198`）。

**请求脱敏子流程（`secureRequestDump`，`recovery.go:98`）**

- `httputil.DumpRequest(r, false)` 导出请求（不带 body），按行拆分后把 `Authorization:` 行替换为 `Authorization: *`（`recovery.go:100-105`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| 输出 Writer | `Recovery()` 用 `DefaultErrorWriter`（默认 `os.Stderr`） | `recovery.go:36` |
| `recovery ...RecoveryFunc` | 可选变参；不传则用 `defaultHandleRecovery` | `recovery.go:45-50` |
| 日志前缀 | `"\n\n\x1b[31m"` 红色 ANSI + `LstdFlags` | `recovery.go:56` |
| 栈跳过帧数 | `stackSkip = 3` | `recovery.go:28` |
| Debug 模式详情 | 仅 `IsDebugging()` 时打印完整请求 dump | `recovery.go:73` |

## 5. 错误与重试语义

- panic 被 `recover()` 捕获后**不会**再向上传播；本中间件是请求 goroutine 的最后一道防线。
- broken pipe（客户端断连）不返回 500，只 `Abort()` 并 `c.Error(err)`，避免往死连接写状态码。
- `readNthLine` 打开源码文件失败时 `continue` 跳过该帧（`recovery.go:136-138`），不影响其余栈帧输出。
- 无重试、无退避：Gin 中间件模型不支持自动重放请求；恢复即终止本次调用链。
- `c.Error` 对 nil 会 panic（`context.go:263-265`），故 `defaultHandleRecovery` 先把非 error 的 `rec` 包成 error（`recovery.go:110-112`）。

## 6. 并发细节

- 本中间件不新建 goroutine；`defer/recover` 在当前请求 goroutine 内同步执行。
- 每个 panic 现场调用一次 `runtime.Caller` 循环与 `os.Open`（读源码行），属重操作但仅异常路径触发，不影响热路径。
- `log.New` 创建的 logger 在闭包外构造一次，闭包内仅调用其方法；`log.Logger` 自身带互斥锁，并发写 stderr 安全。
- 无 context 取消传播概念——panic 即终止，不走 `ctx.Done()`。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `recovery.go` 全部：恢复中间件工厂、broken pipe 判定、请求脱敏、栈帧格式化、默认恢复动作。

**Out-of-Scope（不在本仓库源码内）**
- `runtime`、`syscall`、`net/http/httputil`、`log`、`bufio` 等标准库：栈展开、错误比较、请求 dump，外部依赖。
- `internal/bytesconv`（`recovery.go:23`）：`BytesToString` 零拷贝，见 internal-utils 叶子。
- `Context.AbortWithStatus/Error` 实现在 `context.go`（context-object 叶子）。
- 本叶子不负责把错误上报外部监控系统（那是用户自定义 `RecoveryFunc` 的事）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`Engine.Default()` 在 `gin.go:239` 把 `Recovery()` 挂在 `Logger()` 之后；中间件顺序保证 panic 发生时 Recovery 的 defer 已注册。
- 本叶子 → 下游：`c.Next()`（context-object 叶子）驱动业务 handler。
- 本叶子 → 相邻：`c.Error` 把 panic 写入 `c.Errors`（error-model 叶子）；`secureRequestDump` 用 `internal/bytesconv`（internal-utils 叶子）。
- 输出 → 本叶子：`DefaultErrorWriter` 由 runtime-mode 叶子定义。

## 9. 语言专项适配口径

- **Go 并发模型**：利用 `defer/recover` 在 per-request goroutine 内兜底，是 Go 唯一合法的 panic 恢复点；不涉及 channel/workqueue。
- **栈自省**：`runtime.Caller` + `runtime.FuncForPC` + 手动读源码行，复刻 `runtime/debug.Stack` 的可读格式，但额外打印触发行源码。
- **错误比较**：用 `errors.Is(err, syscall.EPIPE)` 而非 `==`，兼容 wrapped error 链。
- 无 K8s 控制器模式；纯 HTTP 中间件。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Recovery 中间件架构图 | `recovery-middleware-architecture.html` | architecture | showcase |
| panic 主路径时序图 | `recovery-middleware-sequence.html` | sequence | showcase |
| JSON IR | `json/recovery-middleware-architecture.json`、`json/recovery-middleware-sequence.json` | — | — |

补 sequence 的理由：panic→recover→broken pipe 判定→500 是跨多个参与者的时序流程，文字调用链与时序图互证。
