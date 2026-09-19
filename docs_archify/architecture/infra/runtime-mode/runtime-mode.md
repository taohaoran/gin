# 运行时模式与全局输出（runtime-mode）

> 本文是 `infra` 域下的叶子子系统文档。域级总览见 `../infra.md`，
> 本文只展开 debug/release/test 三态模式与全局 Writer，不重复 debug 打印细节（见 debug-print 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `mode.go`（101 行）、`deprecated.go`（24 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 模式常量 | `DebugMode/ReleaseMode/TestMode` 字符串 | `mode.go:21-25` |
| `SetMode(value)` | 按字符串切换模式，原子存储 | `mode.go:58` |
| `Mode()` | 返回当前模式名字符串 | `mode.go:98` |
| `IsDebugging()` | 原子读 `ginMode == debugCode` | `debug.go:22` |
| 环境变量 `GIN_MODE` | 包 init 自动读取 | `mode.go:17`、`mode.go:52-55` |
| `DefaultWriter` | 全局调试/日志输出 Writer（默认 stdout） | `mode.go:42` |
| `DefaultErrorWriter` | 全局错误输出 Writer（默认 stderr） | `mode.go:45` |
| 绑定/解码器开关 | `DisableBindValidation`、`EnableJsonDecoderUseNumber/DisallowUnknownFields` | `mode.go:81`、`mode.go:87`、`mode.go:93` |
| 遗留 API | `BindWith` 已废弃，提示迁移到 Must/ShouldBindWith | `deprecated.go:17` |

对外暴露点：`SetMode` 是用户控制生产行为（关调试输出）的主入口；`GIN_MODE=release` 环境变量零代码生效。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `ginMode` | `mode.go:48` | `int32`，初值 `debugCode`；原子读写 |
| `modeName` | `mode.go:49` | `atomic.Value`，存当前模式字符串 |
| `debugCode/releaseCode/testCode` | `mode.go:29-32` | iota 三态内部码（0/1/2） |
| `DefaultWriter/DefaultErrorWriter` | `mode.go:42`、`mode.go:45` | `io.Writer` 全局变量，用户可替换 |
| `EnvGinMode` | `mode.go:17` | 常量 `"GIN_MODE"` |

设计要点：对外暴露稳定字符串 `debug/release/test`，内部用 `int32` 码做快速比较，`atomic.Value` 存可读名。

## 3. 关键调用链

**主路径：包初始化自动定模式（`init()`，`mode.go:52`）**

1. 包加载时 `os.Getenv(EnvGinMode)` 读环境变量（`mode.go:53`）。
2. 直接 `SetMode(mode)`（`mode.go:54`）——空串也交给 `SetMode` 处理。

**模式切换主路径（`SetMode`，`mode.go:58`）**

1. 空值兜底：若 `flag.Lookup("test.v") != nil`（即 `go test`）自动 `TestMode`，否则 `DebugMode`（`mode.go:59-65`）。
2. switch 三分支，分别 `atomic.StoreInt32(&ginMode, debugCode/releaseCode/testCode)`（`mode.go:67-76`）。
3. 未知字符串直接 `panic("gin mode unknown: ...")`（`mode.go:75`）。
4. 最后 `modeName.Store(value)` 保存可读名（`mode.go:77`）。

**运行期读模式（`IsDebugging`，`debug.go:22`）**

- `atomic.LoadInt32(&ginMode) == debugCode`，决定 debug 打印/Recovery 是否输出详细信息。

**绑定开关转发（`DisableBindValidation` 等）**

- 直接改 `binding` 包的全局变量：`binding.Validator = nil`（`mode.go:82`）、`binding.EnableDecoderUseNumber = true`（`mode.go:88`）——本层只做转发，定义在 binding 包。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `GIN_MODE` 环境变量 | 未设时 init 走 SetMode 空值逻辑 | `mode.go:53` |
| 空值 + `go test` | 自动 `TestMode` | `mode.go:60-61` |
| 空值 + 普通运行 | `DebugMode` | `mode.go:63` |
| 未知模式值 | panic | `mode.go:75` |
| `DefaultWriter` | `os.Stdout`；可用 `go-colorable` 替换支持 Windows 彩色 | `mode.go:42` |
| `DefaultErrorWriter` | `os.Stderr` | `mode.go:45` |

## 5. 错误与重试语义

- 模式配置错误（未知字符串）在**启动期** panic，fail-fast，不继续运行。
- `deprecated.go` 的 `BindWith` 不返回新错误，而是 `log.Println` 一条迁移提示后委托 `c.MustBindWith`（`deprecated.go:18-22`），属向后兼容垫片。
- 无重试；模式切换是幂等的，可随时重设。

## 6. 并发细节

- 不创建 goroutine。
- `ginMode int32` 用 `atomic.StoreInt32/LoadInt32`（`mode.go:69`、`debug.go:23`），`modeName atomic.Value` 用 `Store/Load`（`mode.go:77`、`mode.go:99`）——保证路由注册期与请求期并发读模式无数据竞争。
- `DefaultWriter/DefaultErrorWriter` 是普通 `io.Writer` 变量，Gin 约定在初始化期赋值，运行期不热切换。
- 无 context 传播。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `mode.go`：模式定义、SetMode、全局 Writer、绑定开关转发。
- `deprecated.go`：遗留 `BindWith` 垫片。

**Out-of-Scope（不在本仓库源码内）**
- `binding` 包（`mode.go:13`）：`Validator`/`EnableDecoderUseNumber` 等实际变量定义，见 binding-registry 叶子。
- `flag`/`os` 标准库：测试模式探测、环境变量，外部依赖。
- debug 打印的具体格式化在 debug-print 叶子。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`Engine.New()`/`Default()` 在启动时调用 `debugPrintWARNINGNew/Default`（debug-print 叶子），后者读 `IsDebugging`。
- 本叶子 → 下游：`SetMode` 控制 debug-print、Recovery（`IsDebugging()` 分支）、Logger 的输出行为。
- 本叶子 → 相邻：`DefaultWriter/ErrorWriter` 被 Logger/Recovery/debugPrint 写入；开关转发到 binding 包。

## 9. 语言专项适配口径

- **Go 并发模型**：`int32` + `atomic` + `atomic.Value` 是 Gin 跨包共享全局开关的标准做法，避免 mutex 的读放大；`go test` 下自动 TestMode 利用 `flag.Lookup` 探测测试二进制。
- **库形态**：模式是包级全局状态，`ginS` 门面（global-singleton 叶子）与各中间件共享同一份。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 运行时模式架构图 | `runtime-mode-architecture.html` | architecture | standard（showcase 因纵向连线布局回退 standard） |
| JSON IR | `json/runtime-mode-architecture.json` | — | — |

本叶子不补时序图：模式切换为"读 env→原子写→各读取点"的简单数据流，架构图已表达。
