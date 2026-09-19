# debug-print 路由调试输出（debug-print）

> 本文是 `builtin-middleware` 域下的叶子子系统文档。域级总览见 `../builtin-middleware.md`，
> 本文只展开调试期的路由注册打印与模板加载提示，不重复运行模式切换（见 runtime-mode 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `debug.go`（115 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `debugPrintRoute` | 注册路由时打印 `方法 路径 --> handler (N handlers)` | `debug.go:32` |
| `debugPrint` | 通用调试打印，带 `[GIN-debug]` 前缀 | `debug.go:56` |
| `debugPrintLoadTemplate` | 打印已加载 HTML 模板名清单 | `debug.go:44` |
| `debugPrintWARNINGDefault` | 提示 `Default()` 已挂 Logger+Recovery + Go 版本警告 | `debug.go:81` |
| `debugPrintWARNINGNew` | 提示 debug 模式应在生产切 release | `debug.go:92` |
| `debugPrintWARNINGSetHTMLTemplate` | 提示 `SetHTMLTemplate` 非线程安全 | `debug.go:100` |
| `debugPrintError` | 向 `DefaultErrorWriter` 打印 `[GIN-debug] [ERROR]` | `debug.go:110` |
| `DebugPrintRouteFunc` | 可整体替换的路由打印格式钩子变量 | `debug.go:27` |
| `DebugPrintFunc` | 可整体替换的通用调试打印钩子变量 | `debug.go:30` |
| Go 版本门槛 | `ginSupportMinGoVer = 25`，运行时版本检测 | `debug.go:16`、`debug.go:72` |

对外暴露点：路由注册路径 `debugPrintRoute` 被 `RouterGroup.handle` / 路由树在 `addRoute` 时调用。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `debugPrintRouteFunc` 变量 | `debug.go:27` | 签名 `func(httpMethod, absolutePath, handlerName string, nuHandlers int)`，用户可替换 |
| `DebugPrintFunc` 变量 | `debug.go:30` | 签名 `func(format string, values ...any)`，替换 `[GIN-debug]` 输出 |
| `ginSupportMinGoVer` | `debug.go:16` | 常量 25，要求 Go 1.25+ |
| `runtimeVersion` | `debug.go:18` | 包级变量 `runtime.Version()` |
| `getMinVer` | `debug.go:72` | 从版本字符串提取次版本号（如 `go1.26.0`→26） |

扩展点：两个包级函数变量 `DebugPrintRouteFunc`/`DebugPrintFunc` 是用户重定向调试输出的缝——置 nil 用默认格式，置非 nil 整体接管。

## 3. 关键调用链

**主路径：注册一条路由时打印路由表（`debugPrintRoute`，`debug.go:32`）**

1. 调用方传入 `httpMethod/absolutePath/handlers HandlersChain`；先 `if IsDebugging()` 判定（`debug.go:33`；`IsDebugging` 在 `debug.go:22`，读原子 `ginMode`）。
2. 取 `nuHandlers := len(handlers)`，并用 `nameOfFunction(handlers.Last())` 取末个 handler 函数名（`debug.go:34-35`；`nameOfFunction` 在 `utils.go:132`，用 `runtime.FuncForPC`）。
3. 若 `DebugPrintRouteFunc == nil`，走默认格式 `debugPrint("%-6s %-25s --> %s (%d handlers)\n", ...)`（`debug.go:37`）；否则调用用户钩子 `DebugPrintRouteFunc(...)`（`debug.go:39`）。

**通用打印子流程（`debugPrint`，`debug.go:56`）**

- 非 debug 模式直接 `return`（`debug.go:57-59`）。
- `DebugPrintFunc` 非 nil 则转交用户钩子（`debug.go:61-64`）。
- 补全尾换行后 `fmt.Fprintf(DefaultWriter, "[GIN-debug] "+format, ...)`（`debug.go:66-69`）。

**Go 版本告警（`debugPrintWARNINGDefault`，`debug.go:81`）**

- `getMinVer(runtimeVersion)` 解析次版本号（`debug.go:82`；`getMinVer` 在 `debug.go:72` 截取首尾两个 `.` 之间的数字）。
- 若 `< ginSupportMinGoVer(25)` 打印 `[WARNING] Now Gin requires Go 1.25+`（`debug.go:83`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| 仅 Debug 模式 | 所有 `debugPrint*` 先查 `IsDebugging()`，release/test 静默 | `debug.go:33`、`debug.go:57`、`debug.go:111` |
| `DebugPrintRouteFunc` | nil；非 nil 时路由打印整体走用户格式 | `debug.go:36-40` |
| `DebugPrintFunc` | nil；非 nil 时 `debugPrint` 整体走用户格式 | `debug.go:61-64` |
| 输出目标 | 默认 `DefaultWriter`（stdout）；`debugPrintError` 用 `DefaultErrorWriter`（stderr） | `debug.go:69`、`debug.go:112` |
| Go 版本门槛 | `ginSupportMinGoVer = 25` | `debug.go:16` |

## 5. 错误与重试语义

- 调试打印本身不返回错误；`debugPrintError` 只把 `[GIN-debug] [ERROR] %v` 写到 stderr（`debug.go:112`）。
- 路由冲突等真正错误由路由树 panic（见 tree.go），本叶子只负责在 debug 模式登记"发生了什么"。
- 无重试；纯打印。

## 6. 并发细节

- 不创建 goroutine。
- `IsDebugging()` 用 `atomic.LoadInt32(&ginMode)`（`debug.go:23`），与 `SetMode` 的原子写配对，路由注册（可能并发）与请求期读模式无数据竞争。
- 两个替换钩子变量是普通包级变量；Gin 约定它们只在初始化期赋值，请求期不读写。
- `fmt.Fprintf(DefaultWriter, ...)` 写 stdout，`log`/fmt 对同目标的并发写按行原子性有限，但仅 debug 路径，可接受。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `debug.go` 全部：路由/模板调试打印、告警函数、Go 版本检测、替换钩子。

**Out-of-Scope（不在本仓库源码内）**
- `runtime`、`html/template`、`sync/atomic` 标准库：版本、模板遍历、原子操作，外部依赖。
- `DefaultWriter/DefaultErrorWriter`、`IsDebugging/SetMode` 属 runtime-mode 叶子。
- `nameOfFunction` 在 `utils.go`（工具函数）。
- 本叶子不实现路由匹配逻辑（tree.go / routergroup 叶子）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`RouterGroup` 注册路由（routergroup 叶子）时调用 `debugPrintRoute`；`Engine.SetHTMLTemplate` 加载模板时调用 `debugPrintLoadTemplate`。
- 本叶子 → 相邻：读取 `IsDebugging` 与 `ginMode`（runtime-mode 叶子）；写入 `DefaultWriter/DefaultErrorWriter`（runtime-mode 叶子）。
- 本叶子 → 下游：无业务下游，仅输出到 Writer。

## 9. 语言专项适配口径

- **Go 并发模型**：`ginMode int32` + `atomic.LoadInt32` 是 Gin 跨包共享运行模式的并发安全做法（runtime-mode 叶子详述）。
- **反射自省**：`runtime.FuncForPC` 取 handler 名用于调试展示，属只读自省。
- **库形态**：调试输出由 `DebugPrintFunc` 钩子重定向，便于测试期捕获。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| debug-print 架构图 | `debug-print-architecture.html` | architecture | showcase |
| JSON IR | `json/debug-print-architecture.json` | — | — |

本叶子不补时序图：注册→判模式→格式化→输出为单线程线性流程，已在第 3 节文字化。
