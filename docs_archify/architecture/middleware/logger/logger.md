# 日志中间件（logger）

> 本文是 `middleware` 域下的叶子子系统文档。域级总览见 [`../middleware.md`](../middleware.md)。
> 本文展开 gin 内置访问日志中间件的输出协议与配置；中间件链如何被 `c.Next()` 驱动见
> [`../middleware-core/middleware-core.md`](../middleware-core/middleware-core.md)；日志里读取的
> 错误聚合见 [`../../support/errors/errors.md`](../../support/errors/errors.md)。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 默认访问日志中间件 | `Logger()` 以默认配置产出 `HandlerFunc`，写入 `gin.DefaultWriter`（默认 `os.Stdout`） | `logger.go:224` |
| 自定义格式化器 | `LoggerWithFormatter(f)` 注入用户自定义 `LogFormatter` | `logger.go:229` |
| 自定义输出目标 | `LoggerWithWriter(out, notlogged...)` 写指定 `io.Writer` 并跳过若干路径 | `logger.go:237` |
| 全量配置入口 | `LoggerWithConfig(LoggerConfig)` 统一兜底 Formatter/Output/SkipPaths/SkipQueryString/Skip | `logger.go:245` |
| 日志参数字段协议 | `LogFormatterParams` 把请求后时刻的时间戳、状态码、耗时、客户端 IP、方法、路径、错误、响应体大小打包给格式化器 | `logger.go:68` |
| 终端 ANSI 配色 | `StatusCodeColor/LatencyColor/MethodColor` 按状态码/耗时/方法返回 ANSI 色码；`IsOutputColor` 判定是否着色 | `logger.go:94`、`logger.go:112`、`logger.go:133`、`logger.go:162` |
| 默认格式化串 | `defaultLogFormatter` 输出 `[GIN] 时间 | 状态 | 耗时 | IP | 方法 路径` 一行 | `logger.go:167` |
| 控制台颜色模式控制 | `DisableConsoleColor/ForceConsoleColor` 切换全局 `consoleColorMode` | `logger.go:197`、`logger.go:202`、`logger.go:36` |
| 错误日志中间件 | `ErrorLogger/ErrorLoggerT`：`c.Next()` 后把 `c.Errors` 中匹配类型的错误以 JSON 写出 | `logger.go:207`、`logger.go:212` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `LoggerConfig` | `logger.go:39` | 配置结构体：`Formatter`、`Output`、`SkipPaths`、`SkipQueryString`、`Skip`，全字段 optional |
| `Skipper` | `logger.go:62` | `func(c *Context) bool`，返回 true 则跳过本次日志 |
| `LogFormatter` | `logger.go:65` | `func(LogFormatterParams) string`，格式化器签名 |
| `LogFormatterParams` | `logger.go:68` | 输出协议数据包：`Request/TimeStamp/StatusCode/Latency/ClientIP/Method/Path/ErrorMessage/BodySize/Keys` 及私有 `isTerm` |
| `consoleColorModeValue` | `logger.go:17` | `autoColor/disableColor/forceColor` 三态；包级变量 `consoleColorMode`（`logger.go:36`） |
| `ErrorLoggerT` | `logger.go:212` | 错误聚合输出中间件工厂，按 `ErrorType` 过滤 `c.Errors` |

## 3. 关键调用链

### 3.1 一次请求的访问日志产出

1. 链首的 Logger 闭包（`LoggerWithConfig` 返回，`logger.go:275`）进入后先 `start := time.Now()`，记录 `path := c.Request.URL.Path` 与 `raw := c.Request.URL.RawQuery`（`logger.go:277-279`）。
2. 调用 `c.Next()`（`logger.go:282`）**同步执行下游整条中间件与业务链**——这是 Logger 能拿到最终状态码和耗时的关键：本函数栈帧挂起，直到所有下游处理器跑完才返回。
3. 返回后先判断是否跳过：命中 `skip[path]`（由 `SkipPaths` 预处理成 map，`logger.go:267`）或 `conf.Skip(c)` 为真则直接 return，不写日志（`logger.go:285`）。
4. 否则填充 `LogFormatterParams`：`TimeStamp=time.Now()`、`Latency=TimeStamp.Sub(start)`、`ClientIP=c.ClientIP()`、`Method=c.Request.Method`、`StatusCode=c.Writer.Status()`、`ErrorMessage=c.Errors.ByType(ErrorTypePrivate).String()`、`BodySize=c.Writer.Size()`（`logger.go:296-304`）。
5. 若 `raw != "" && !conf.SkipQueryString`，把 `?raw` 拼回 path（`logger.go:306`），最后 `fmt.Fprint(out, formatter(param))`（`logger.go:312`）输出到 `Output`。

### 3.2 终端颜色判定

1. `LoggerWithConfig` 在装配期探测输出是否终端：`out.(*os.File)` 失败、`TERM=dumb`、或 `isatty.IsTerminal/IsCygwinTerminal` 均为 false 时 `isTerm=false`（`logger.go:260`）。
2. 格式化时 `IsOutputColor()`（`logger.go:162`）：`consoleColorMode==forceColor` 强制着色；`autoColor` 时仅当 `isTerm` 才着色；非终端（管道/文件）自动无色。
3. `defaultLogFormatter`（`logger.go:167`）据此取 `StatusCodeColor/LatencyColor/MethodColor` 的 ANSI 前缀，并按耗时量级 `Truncate` 截断后格式化。

### 3.3 错误日志中间件

1. `ErrorLogger()`（`logger.go:207`）= `ErrorLoggerT(ErrorTypeAny)`（`logger.go:212`）。
2. 返回闭包先 `c.Next()`，再 `c.Errors.ByType(typ)` 过滤（`logger.go:215`）；若有匹配错误则 `c.JSON(-1, errors)` 输出——`-1` 状态码表示不改变已写状态（`logger.go:217`）。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|---|---|---|
| `LoggerConfig.Formatter` | 为 nil 时兜底 `defaultLogFormatter` | `logger.go:40`、`logger.go:247` |
| `LoggerConfig.Output` | 为 nil 时兜底 `DefaultWriter`（包级，默认 `os.Stdout`） | `logger.go:45`、`logger.go:252` |
| `LoggerConfig.SkipPaths` | 路径数组，装配成 `map[string]struct{}` 精确匹配跳过 | `logger.go:49`、`logger.go:267` |
| `LoggerConfig.SkipQueryString` | 默认 false：日志 path 后拼接 `?rawquery`；true 时丢弃查询串（防 API key 泄漏） | `logger.go:54`、`logger.go:306` |
| `LoggerConfig.Skip` | 可选 `Skipper` 函数，按 `*Context` 动态跳过 | `logger.go:58`、`logger.go:285` |
| `consoleColorMode` | 默认 `autoColor`；`DisableConsoleColor()`/`ForceConsoleColor()` 切换 | `logger.go:36`、`logger.go:197` |

## 5. 错误与重试语义

- Logger 本身**不产生业务错误**：它只是观测者，写日志失败（`fmt.Fprint` 到坏 writer）被静默忽略，不影响请求响应。
- 它消费下游错误：`ErrorMessage` 字段取 `c.Errors.ByType(ErrorTypePrivate).String()`（`logger.go:302`）——只把**私有错误**汇总进行日志，公开错误（`ErrorTypePublic`）留给响应体。
- `ErrorLoggerT` 中间件在 `c.Next()` 后若错误非空才写 JSON；无错误时零输出。
- 无重试/退避：纯同步观测，不涉及网络调用。

## 6. 并发细节

- Logger 闭包**无 goroutine、无 channel**：每次请求在独立 `Context` 上同步执行，`start`/`param` 都是栈上局部变量，天然并发安全。
- 全局状态仅 `consoleColorMode`（`logger.go:36`）一个包级变量，由 `Disable/ForceConsoleColor` 在启动期设置；请求期只读，不做热切换，无锁竞争问题。
- `skip` map 在 `LoggerWithConfig` 装配期一次性构建（`logger.go:267`），请求期只读查找。
- 不持有 `c.Keys` 写锁：只读 `c.Keys` 填入 `param.Keys`（`logger.go:292`）；但注意 `LogFormatterParams.Keys` 直接引用 `c.Keys` map，格式化器若异步持有需自行加锁（框架不担保）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `logger.go` 全部：`Logger/LoggerWithFormatter/LoggerWithWriter/LoggerWithConfig`、`LogFormatterParams` 及其颜色方法、`defaultLogFormatter`、`ErrorLogger/ErrorLoggerT`、颜色模式控制。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/mattn/go-isatty`（外部依赖）：判定文件描述符是否终端，不在本仓库源码内。
- `gin.DefaultWriter/DefaultErrorWriter` 的定义与模式（debug/release/test）在 `mode.go`，属配置域。
- `c.Errors` 的类型化聚合（`ByType/ErrorTypePrivate`）在 support 域 errors 叶子。
- `c.Writer.Status()/Size()` 的响应写入统计在 `response_writer.go`，属响应域。

## 8. 与相邻子系统交互

- **上游**：链协议 `c.Next()`（middleware-core）把 Logger 放在链首，先计时后放行下游。
- **下游读取**：Logger 读 `c.Writer.Status()/Size()`（响应域）、`c.Errors.ByType`（support/errors）、`c.ClientIP()`（上下文客户端 IP 解析）。
- **输出**：写 `io.Writer`（默认 stdout，外部终端/文件）。
- Recovery 通常与 Logger 一同注册（`Default()` `gin.go:239`），二者都靠 `c.Next()` 包裹下游，互不感知。

## 9. 语言专项适配口径（Go）

- **并发模型**：标准库中间件同步模型，无 goroutine 启停、无 workqueue；并发安全靠"无共享可变状态 + 局部变量"。这与 K8s 控制器模式完全无关。
- **internal 边界**：本叶子不 import `internal/`；颜色用 ANSI 字符串常量（`logger.go:26-33`）。
- **依赖注入式扩展点**：`LogFormatter` 函数值（`logger.go:65`）与 `io.Writer`（`logger.go:45`）是两个主要扩展缝，用户可替换为 Zap/Logrus 输出——符合 Go "接受函数值与接口"的惯用法。
- **多二进制**：纯库函数，无独立二进制。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 日志中间件组件与配置图 | [logger-architecture.html](logger-architecture.html) | architecture | showcase |
| 请求计时日志产出时序图 | [logger-sequence.html](logger-sequence.html) | sequence | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 workflow/dataflow/lifecycle 图——日志产出是单次请求的同步观测（已由 sequence 表达），无多角色泳道流程、无数据 ETL 管道、也无单一实体状态机（颜色三态是包级配置而非请求实体状态），按资源节省原则省略。
