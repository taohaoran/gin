# 路径与工具函数（path-utils）

> 本文是 `router-tree` 域下的叶子子系统文档。域级总览见 `../router-tree.md`。
> 本文只展开 **路径规范化（cleanPath/removeRepeatedChar）、路径拼接（joinPaths）、地址解析（resolveAddress）以及通用小工具（assert1/nameOfFunction/safeInt8 等）**，不重复展开这些函数如何被路由树/Engine 调用（分别见 `radix-tree` 与 `core-engine` 各叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。主源码文件：`path.go`（源自 httprouter）、`utils.go`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 路径规范化 | `cleanPath`：URL 版 `path.Clean`，消除 `.`/`..`、合并重复斜杠、补尾斜杠 | `path.go:23` |
| 惰性缓冲 | `bufApp`：未修改原串时零分配返回子串，必要时才建堆缓冲 | `path.go:128` |
| 去重复字符 | `removeRepeatedChar`：合并连续重复字符（如多斜杠），无重复时原样返回 | `path.go:155` |
| 路径拼接 | `joinPaths`：`path.Join` + 保留相对路径尾斜杠语义 | `utils.go:136` |
| 监听地址解析 | `resolveAddress`：无参读 `PORT` 环境变量，缺省回退 `:8080`，多参 panic | `utils.go:148` |
| 不可达断言 | `assert1(guard, text)`：guard 为假直接 panic（注册期编程错误） | `utils.go:86` |
| 反射取函数名 | `nameOfFunction`：`runtime.FuncForPC(...).Name()`，用于 `Routes()`/HandlerName | `utils.go:132` |
| 安全整数裁剪 | `safeInt8/safeUint16`：超 `MaxInt8/MaxUint16` 时裁剪，用于 countParams/countSections | `utils.go:175`、`utils.go:183` |
| 取末字符 | `lastChar`：空串 panic，否则返回末字节 | `utils.go:125` |
| 内容协商工具 | `filterFlags`（剥 MIME 参数）、`parseAccept`（解析 Accept）、`chooseData` | `utils.go:92`、`utils.go:111`、`utils.go:101` |
| 类型辅助 | `isASCII`（判断纯 ASCII） | `utils.go:165` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `stackBufSize = 128` | `path.go:8` | cleanPath/removeRepeatedChar 的栈上缓冲初值 |
| `cleanPath` | `path.go:23` | 核心路径规范化，`r/w` 双下标原地扫描 |
| `bufApp` | `path.go:128` | 惰性写缓冲，调用被内联，零函数调用开销 |
| `joinPaths` | `utils.go:136` | RouterGroup 拼绝对路径的统一入口 |
| `resolveAddress` | `utils.go:148` | `Run()` 的地址归一 |
| `assert1` | `utils.go:86` | 全注册期共用的"不可达/非法输入"断言 |
| `nameOfFunction` | `utils.go:132` | 路由清单导出时取 handler 函数全限定名 |
| `H` / `BindKey` / `localhostIP` | `utils.go:62`、`utils.go:20`、`utils.go:23` | 快捷 map 类型、绑定键常量、本地回环常量 |

## 3. 关键调用链

**链路 A：cleanPath 规范化（以 RemoveExtraSlash 场景为例）**

1. Engine `handleHTTPRequest` 在 `RemoveExtraSlash=true` 时 `rPath = cleanPath(rPath)`（`gin.go:703-704`）。
2. `cleanPath` 把空串转 `/`，确保路径以 `/` 开头（`path.go:25-52`）。
3. 单遍扫描：`p[r]` 为 `/` 跳过空段；`.` 单元素跳过；`..` 回退到上一个 `/`；真实字符 `bufApp` 写出（`path.go:61-108`）。
4. 处理尾斜杠：若 `trailing` 且 `w>1` 补一个 `/`（`path.go:112-115`）。
5. **零分配优化**：若全程未修改（`len(buf)==0`）直接返回 `p[:w]` 子串；否则返回 `string(buf[:w])`（`path.go:120-123`）。`bufApp` 在"下一字符与原串相同"时连缓冲都不建（`path.go:134-148`）。

**链路 B：removeRepeatedChar（redirectTrailingSlash 用）**

1. `redirectTrailingSlash` 对 `X-Forwarded-Prefix` 处理时 `removeRepeatedChar(prefix, '/')`（`gin.go:786`）。
2. 先扫描一次判断是否真有连续重复，没有则原样返回，避免任何分配（`path.go:157-166`）。
3. 有重复时 `bufApp` 写出首个字符、跳过后续连续同字符，非该字符正常拷贝（`path.go:177-194`）。

**链路 C：joinPaths 与 resolveAddress**

- `RouterGroup.calculateAbsolutePath = joinPaths(group.basePath, relativePath)`（`routergroup.go:260-261`）：`path.Join` 拼接后，若 relativePath 以 `/` 结尾而结果未以 `/` 结尾，则补一个 `/`（`utils.go:141-145`），保证子组尾斜杠语义。
- `Run()` 调 `resolveAddress(addr)`（`gin.go:548`）：无参读 `PORT` 环境变量补 `:` 前缀，未设则回退 `:8080`，多于 1 个参数 panic（`utils.go:148-161`）。

## 4. 配置项

| 配置项 / 常量 | 默认 / 行为 | 位置 |
|------|------|------|
| `stackBufSize` | 128，栈上缓冲初值，超出才堆分配 | `path.go:8` |
| `PORT` 环境变量 | `Run()` 无参时读，补 `:` 前缀 | `utils.go:151-153` |
| 缺省端口 | 未设 `PORT` 时回退 `:8080` | `utils.go:155-156` |
| `localhostIP/localhostIPv6` | `127.0.0.1` / `::1` 常量 | `utils.go:23`、`utils.go:26` |

## 5. 错误与重试语义

- **assert1 panic**：`assert1` 是"不应发生"断言，guard 为假即 panic（如 `addRoute` 里路径必须以 `/` 开头、method 非空、至少一个 handler，见 `gin.go:365-367`）——开发期 fail-fast，非运行期可恢复错误。
- **lastChar panic**：对空串调用直接 panic（`utils.go:126-128`）。
- **resolveAddress panic**：传入多于 1 个地址参数时 panic（`utils.go:160`）。
- **无错误返回**：cleanPath/removeRepeatedChar/joinPaths 均不返回 error，只返回规范化后的字符串；这是纯函数式设计。
- **无重试**：纯字符串处理，不涉及重试。

## 6. 并发细节

- **纯函数无共享状态**：本叶子全部函数为无状态纯函数（除 `resolveAddress` 读环境变量），可安全并发调用，无需锁。
- **零分配/少分配**：cleanPath 用 `r/w` 双下标原地扫描 + `bufApp` 惰性缓冲，常见路径（无需清理）零分配——与路由树零分配设计呼应。
- **`bufApp` 内联**：注释说明调用被编译器内联，避免函数调用开销（`path.go:127-128`）。
- **runtime 反射**：`nameOfFunction` 用 `runtime.FuncForPC` 取函数名，仅在 `Routes()` 导出/调试时调用，不在请求热路径。
- **context.Context**：本叶子不涉及。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `path.go`：路径规范化与去重。
- `utils.go`：地址解析、断言、反射、整数裁剪、内容协商小工具。

**Out-of-Scope（不在本仓库源码内）**
- Go 标准库 `path.Join`、`os.Getenv`、`runtime.FuncForPC`、`unicode`/`utf8`。
- httprouter 上游算法版权（path.go 头部 BSD-3 声明）。

**不做什么**：不做路由匹配（radix-tree）、不做 HTTP 派发（core-engine）、不做模板/渲染。

## 8. 与相邻子系统交互

- **radix-tree / Engine → path-utils**：`cleanPath` 被 `RemoveExtraSlash` 预处理调用；`countParams/countSections` 经 `safeInt8` 裁剪。
- **router-group → path-utils**：`calculateAbsolutePath` 调 `joinPaths`。
- **Engine → path-utils**：`Run` 调 `resolveAddress`；`redirectTrailingSlash` 调 `removeRepeatedChar`/`sanitizePathChars`。
- **core-engine → path-utils**：`addRoute`/`combineHandlers` 用 `assert1`；`Routes()` 用 `nameOfFunction`。
- **Context 渲染族 → path-utils**：内容协商用 `filterFlags`/`parseAccept`/`chooseData`。

## 9. 语言专项适配口径

- **纯函数 + 零分配**：Go 高性能字符串处理范式——`r/w` 双下标 + 惰性缓冲（`bufApp`），避免在热路径上 `strings.Builder`/`[]byte` 分配。
- **assert1 = Go 的 invariant**：用 panic 表达"注册期不变量"，区别于运行期 error；这与 K8s 控制器的"运行期 status 反映错误"不同——本框架把错误前移到启动期。
- **runtime 反射受限使用**：`nameOfFunction` 仅用于调试/导出，不在请求热路径，规避反射开销。
- **internal 边界**：本叶子不引用 internal 包；`cleanPath` 直接操作 `string` 下标。
- **单二进制库形态**：纯工具函数集合，无独立入口。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 路径工具函数调用架构图 | `path-utils-architecture.html` | architecture | 见渲染结果 |

- JSON IR 源文件位于 `json/path-utils-architecture.json`。
- 本叶子**不补数据流图**：理由——cleanPath 是单遍 `r/w` 扫描的纯字符串处理，用第 3 节文字化描述即可，无多阶段管道或状态流转，dataflow 价值低。
