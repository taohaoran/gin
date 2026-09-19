# Logger 中间件（logger-middleware）

> 本文是 `builtin-middleware` 域下的叶子子系统文档。域级总览见 `../builtin-middleware.md`，
> 本文只展开请求访问日志中间件的实现，不重复 Recovery、BasicAuth 等其它中间件（见各自叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `logger.go`（315 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Logger()` 便捷入口 | 以默认配置（输出到 `DefaultWriter`、默认 formatter）实例化日志中间件 | `logger.go:224` |
| `LoggerWithConfig(conf)` | 可配置 formatter / 输出 / 跳过路径 / 跳过函数的核心实现 | `logger.go:245` |
| `LoggerWithFormatter(f)` | 仅自定义日志格式函数 | `logger.go:229` |
| `LoggerWithWriter(out, paths...)` | 自定义输出 `io.Writer` 与不记录的路径 | `logger.go:237` |
| 默认格式 `defaultLogFormatter` | 时间/状态码/延迟/客户端IP/方法/路径/错误单行格式 | `logger.go:167` |
| ANSI 彩色辅助方法 | `StatusCodeColor/LatencyColor/MethodColor/IsOutputColor` | `logger.go:94`、`logger.go:112`、`logger.go:133`、`logger.go:162` |
| 彩色模式控制 | `DisableConsoleColor/ForceConsoleColor` 全局开关 | `logger.go:197`、`logger.go:202` |
| `ErrorLogger/ErrorLoggerT` | 把 `c.Errors` 按类型序列化为 JSON 的中间件 | `logger.go:207`、`logger.go:212` |

对外暴露点：`Logger()` 是 `gin.Default()` 默认挂载的两个中间件之一（见 `gin.go:239`）。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `LoggerConfig` | `logger.go:39` | 日志中间件配置：`Formatter`、`Output`、`SkipPaths`、`SkipQueryString`、`Skip` |
| `Skipper` | `logger.go:62` | `func(c *Context) bool`，按上下文动态决定是否跳过日志 |
| `LogFormatter` | `logger.go:65` | 日志格式函数签名 `func(LogFormatterParams) string` |
| `LogFormatterParams` | `logger.go:68` | 交给 formatter 的全部字段（请求/时间戳/状态码/延迟/ClientIP/方法/路径/错误/BodySize/Keys） |
| `consoleColorMode` | `logger.go:36` | 包级变量 `autoColor/disableColor/forceColor`，控制彩色输出 |
| `defaultLogFormatter` | `logger.go:167` | 包级默认格式函数变量（可被整体替换） |

扩展点：`LoggerConfig.Formatter`（替换格式）、`LoggerConfig.Skip`（动态跳过）、`LoggerConfig.Output`（重定向输出）三者均为用户可注入的回调/Writer。

## 3. 关键调用链

**主路径：一次请求访问日志的产生（`LoggerWithConfig` 返回的闭包）**

1. 中间件被调用，先 `start := time.Now()` 并捕获 `path := c.Request.URL.Path`、`raw := c.Request.URL.RawQuery`（`logger.go:277-279`）。
2. 调用 `c.Next()` 把控制权交给下游处理链（`logger.go:282`；`c.Next()` 实现在 `context.go:198`，自增 `handlers` 下标顺序执行）。
3. 下游返回后，命中跳过规则则直接 `return` 不写日志：`skip[path]` 命中或 `conf.Skip(c)` 为真（`logger.go:285`）。
4. 组装 `LogFormatterParams`：`TimeStamp/Latency` 计时差值（`logger.go:296-297`）、`ClientIP()`（`logger.go:299`，实现在 `context.go:998`）、`Writer.Status()`、`Errors.ByType(ErrorTypePrivate).String()`（`logger.go:301-302`）。
5. 若 `raw != "" && !SkipQueryString`，把查询串拼回路径（`logger.go:306-308`），最后 `fmt.Fprint(out, formatter(param))` 输出（`logger.go:312`）。

**构造链：`Logger()` → 真正的闭包**

- `Logger()` 直接 `return LoggerWithConfig(LoggerConfig{})`（`logger.go:225`）。
- `LoggerWithConfig` 在闭包外预先做两件事：解析 `isTerm`（终端探测，`logger.go:260-263`）与把 `SkipPaths` 建成 `map[string]struct{}` 以便 O(1) 命中（`logger.go:265-273`）；这部分在请求到来前完成，避免每请求分配。

**`ErrorLoggerT` 错误收集**：`c.Next()` 后按类型过滤 `c.Errors.ByType(typ)`，非空则 `c.JSON(-1, errors)` 序列化（`logger.go:213-218`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `LoggerConfig.Formatter` | nil 时用 `defaultLogFormatter` | `logger.go:246-249` |
| `LoggerConfig.Output` | nil 时用包级 `DefaultWriter`（默认 `os.Stdout`） | `logger.go:251-254` |
| `LoggerConfig.SkipPaths` | 空切片；构建为 map 精确匹配路径 | `logger.go:265-273` |
| `LoggerConfig.SkipQueryString` | 默认 false，false 时把 `?query` 拼进日志路径 | `logger.go:306` |
| `LoggerConfig.Skip` | nil；非 nil 时每请求调用 `func(c) bool` | `logger.go:285` |
| 终端彩色 | `consoleColorMode=autoColor` 时仅当 `isatty.IsTerminal` 且 `TERM!=dumb` 才上色 | `logger.go:260`、`logger.go:163` |
| 延迟截断 | >1min 截到 10s、>1s 截到 10ms、>1ms 截到 10us | `logger.go:176-183` |

## 5. 错误与重试语义

- 日志中间件本身不产生业务错误，也不重试；它只在下游 `c.Next()` 返回后读取状态。
- 错误文本来自 `c.Errors.ByType(ErrorTypePrivate).String()`（`logger.go:302`），即只聚合私有错误；这意味着 `ErrorTypePublic` 错误不会出现在访问日志行。
- 写日志失败（`fmt.Fprint`）不影响请求本身——输出目标通常是 stdout/文件，错误被静默忽略。
- `ErrorLoggerT` 用 `c.JSON(-1, errors)`：状态码 -1 表示"不改动已写入的状态码"，仅追加 JSON body（`logger.go:217`）。

## 6. 并发细节

- 本中间件**不创建 goroutine**；在 `net/http` 每请求一个 goroutine 的模型下与其它 handler 同帧执行。
- `LoggerWithConfig` 在启动期（闭包外）一次性构建 `skip map` 并固化 `isTerm/formatter/out`，闭包内只读这些不可变量，**无锁**。
- `c.Keys` 由 `Context` 自身的 `sync.RWMutex` 保护（`context.go:286` 写、`context.go:299` 读），日志只读不写。
- `LogFormatterParams.Keys` 直接引用 `c.Keys`（`logger.go:292`），formatter 不应在请求结束后异步持有该引用——`Context` 会被 `sync.Pool` 回收复用。
- context 传播：本中间件不设置 deadline/cancel，仅读取 `c.Request`。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `logger.go` 全部：中间件工厂链、默认 formatter、彩色辅助、终端探测、错误收集中间件。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/mattn/go-isatty`（`logger.go:14`）：终端/TTY 探测，外部依赖。
- `net/http`、`io`、`os`、`time` 标准库：请求对象、Writer、时间，外部依赖。
- `Context` 的 `Next/ClientIP/Errors/Writer` 实现属 `context.go`（context-object 叶子），本叶子只调用。
- 本叶子不实现路由匹配、不实现错误模型定义（见 error-model 叶子）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`Engine.Default()` 在 `gin.go:239` 调用 `Use(Logger(), Recovery())`，把本中间件挂到全局 `Handlers`；`RouterGroup.Use`（routergroup 叶子）负责把它并入每条路由的 handler 链。
- 本叶子 → 下游：调用 `c.Next()`（context-object 叶子）驱动后续 handler。
- 本叶子 → 相邻：读取 `c.Errors`（error-model 叶子）的私有错误做日志；写入 `DefaultWriter`（runtime-mode 叶子的全局 Writer）。

## 9. 语言专项适配口径

- **Go 并发模型**：无 goroutine、无 channel；通过闭包捕获启动期不可变配置（`skip map`/`isTerm`/`formatter`）实现"一次构造、零分配热路径"，符合 Gin 高性能定位。
- **无 K8s 控制器模式**：本叶子是纯 HTTP 中间件，无 Reconcile/informer。
- **库形态**：Gin 是 library，本中间件由使用者通过 `Use()` 装配，不绑定部署形态。
- **依赖方向**：仅依赖 `Context` 抽象与标准库 + isatty，不反向依赖路由树，符合中间件单向依赖请求上下文的设计。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Logger 中间件架构图 | `logger-middleware-architecture.html` | architecture | showcase |
| JSON IR | `json/logger-middleware-architecture.json` | — | — |

本叶子不补时序图/数据流图：单请求主路径为"计时→c.Next()→格式化→输出"的线性流程，已在第 3 节文字化，架构图足以表达组件关系。
