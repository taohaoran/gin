# 错误聚合系统（errors）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 [`../support.md`](../support.md)。
> 本文展开 gin 内置的类型化错误聚合：`c.Errors` 如何收集、分类、过滤与序列化。
> 它被 Logger（`logger.go:302`）、Recovery（`recovery.go:114`）、ErrorLoggerT（`logger.go:215`）共同消费。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 错误类型位标志 | `ErrorType` 是 `uint64` 位掩码：Bind/Render/Private/Public/Any 五类 | `errors.go:16`、`errors.go:18-29` |
| 错误包装结构 | `Error{Err error, Type ErrorType, Meta any}` 把原生 error 加上类型标签与元数据 | `errors.go:32` |
| 链式设置 | `SetType/SetMeta` 返回 `*Error` 支持链式调用 | `errors.go:43`、`errors.go:49` |
| JSON 序列化 | `Error.JSON()` 按 Meta 类型（struct/map/标量）构造输出；`MarshalJSON` 走可插拔 `json.API` | `errors.go:55`、`errors.go:77` |
| error 接口兼容 | `Error()` 委托 `Err.Error()`；`Unwrap()` 支持 `errors.Is/As/Unwrap` | `errors.go:82`、`errors.go:92` |
| 类型判断 | `IsType(flags)` 用位与 `(Type & flags) > 0` | `errors.go:87` |
| 错误切片聚合 | `errorMsgs`（`[]*Error`）：`ByType` 过滤、`Last` 取末位、`Errors()` 转字符串切片、`JSON/String` 批量序列化 | `errors.go:38`、`errors.go:98`、`errors.go:116`、`errors.go:130`、`errors.go:141`、`errors.go:161` |
| 上下文收集入口 | `Context.Error(err)` 用 `errors.As` 把任意 error 包成 `*Error` 追加到 `c.Errors` | `context.go:262` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ErrorType` | `errors.go:16` | `uint64` 位掩码；`ErrorTypeAny = 1<<64 - 1` 表示全部位 |
| `Error` | `errors.go:32` | 实现 `error` 接口（`errors.go:40` 编译期断言 `var _ error = (*Error)(nil)`） |
| `errorMsgs` | `errors.go:38` | 私有切片类型，挂聚合方法；对外不导出，经 `Context.Errors` 暴露 |
| `json.API` | `codec/json/api.go:10` | 包级 `Core` 接口变量，errors 的 `MarshalJSON` 委托给它做实际 JSON 编码 |

## 3. 关键调用链

### 3.1 一个错误如何进入聚合列表

1. 任意中间件/业务处理器调用 `c.Error(err)`（`context.go:262`）：先断言 `err != nil`（否则 panic，`context.go:263`）。
2. `errors.As(err, &parsedError)`（`context.go:268`）尝试把 err 解包成 `*gin.Error`；失败则包一层 `&Error{Err: err, Type: ErrorTypePrivate}`（`context.go:270-273`）——即用户未标注类型的错误默认归为**私有错误**。
3. 追加到 `c.Errors`（`context.go:276`），返回 `*Error` 供调用方链式 `SetType/SetMeta`。

### 3.2 按类型过滤与输出

1. Logger 在请求结束后调 `c.Errors.ByType(ErrorTypePrivate).String()`（`logger.go:302`）——`ByType`（`errors.go:98`）在 `typ==ErrorTypeAny` 时原样返回，否则遍历切片用 `msg.IsType(typ)`（`errors.go:87`，位与）筛选。
2. `ErrorLoggerT`（`logger.go:212`）调 `ByType(typ)` 后若非空 `c.JSON(-1, errors)`——`errorMsgs.MarshalJSON`（`errors.go:157`）调 `a.JSON()`（`errors.go:141`）：0 条返回 nil，1 条返回单对象，多条返回数组。
3. 单条 `Error.JSON()`（`errors.go:55`）：Meta 是 struct 则原样返回（让外部类型自己序列化）；是 map 则展开成键值；否则包成 `{"meta": ...}`；最后兜底补 `{"error": msg.Error()}`。
4. 实际字节编码由 `json.API.Marshal`（`errors.go:78`）完成——具体用标准库/jsoniter/sonic/go-json 取决于编译期 build tag（见 internal-utils 叶子）。

### 3.3 位掩码分类语义

1. `ErrorTypeBind = 1<<63`（`errors.go:20`）、`ErrorTypeRender = 1<<62`（`errors.go:22`）占高位；`ErrorTypePrivate=1<<0`、`ErrorTypePublic=1<<1`（`errors.go:24-26`）占低位。
2. 一个错误可同时打多个位（如 bind 失败 + public），`IsType` 用位与即可匹配任意子集；`ErrorTypeAny=1<<64-1`（`errors.go:28`）所有位为 1，匹配一切。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|---|---|---|
| 错误默认类型 | `c.Error(err)` 未标注时默认 `ErrorTypePrivate` | `context.go:272` |
| JSON 实现 | 由 build tag 决定（`-tags sonic` 等），运行时不可切换 | `codec/json/*.go` |
| 无配置文件 | errors 是纯内存聚合，无 flags/env | — |

## 5. 错误与重试语义

- errors 本身**不产生错误、不重试**：它是错误的"收纳箱"，只提供分类/过滤/序列化。
- 错误分类决定**是否暴露给客户端**：`ErrorTypePrivate` 只进服务端日志（Logger 读它），`ErrorTypePublic` 可进响应体；`Bind/Render` 错误由 binding/render 域标注。
- `Error(nil)` 会 panic（`context.go:263`）——这是编程错误防护，不是运行时失败路径。
- `Unwrap()`（`errors.go:92`）让 `errors.Is/As` 能穿透 `*gin.Error` 拿到底层错误，与标准错误链互操作。

## 6. 并发细节

- `c.Errors` 是每请求 `Context` 上的切片（`context.go:82`），随 Context 经 `sync.Pool` 每请求独占，请求内串行追加，**无需加锁**。
- `Context.reset()`（`context.go:111`）在归还 pool 前 `c.Errors = c.Errors[:0]` 清空，复用底层数组。
- `errorMsgs` 的方法（`ByType/Last/JSON/String`）都是无状态纯函数，只读切片，可安全并发调用（不同请求各自的切片）。
- 无 goroutine/channel；不持有 context.Context。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `errors.go` 全部：`ErrorType` 常量、`Error`、`errorMsgs` 及其方法。
- `context.go:262` 的 `Context.Error` 收集入口。
- `codec/json/api.go` 的 `json.API` 序列化委托点。

**Out-of-Scope（不在本仓库源码内）**
- 各 `ErrorType` 的**标注方**：`ErrorTypeBind` 由 `binding` 包（binding 域）在绑定失败时打标，`ErrorTypeRender` 由 `render` 包打标，本文不展开。
- 具体 JSON 编码器实现（标准库/jsoniter/sonic/go-json）见 support/internal-utils 叶子。
- 错误如何变成 HTTP 响应由 render 域与业务处理器决定。

## 8. 与相邻子系统交互

- **上游（写入方）**：业务处理器 / binding 域 / render 域 / Recovery（`recovery.go:83,114`）调 `c.Error()`。
- **下游（读取方）**：Logger 读 `ByType(ErrorTypePrivate)`（`logger.go:302`）；`ErrorLoggerT` 读 `ByType(typ)` 并 JSON 输出（`logger.go:215`）；业务处理器可读 `c.Errors.Last()` 做自定义响应。
- **横向**：序列化委托 `codec/json.API`（internal-utils 叶子），实现可插拔。

## 9. 语言专项适配口径（Go）

- **错误包装惯用法**：`Error.Unwrap()`（`errors.go:92`）实现 Go 1.13+ 错误链协议，与 `errors.Is/As` 互操作；`Context.Error` 用 `errors.As` 解包（`context.go:268`）——这是 Go 现代错误处理范式。
- **位掩码分类**：用 `uint64` 位标志而非枚举，是为了一个错误可同时属于多类（多态标签），`IsType` 位与查询 O(1)。
- **internal/依赖方向**：errors.go 只 import 自家 `codec/json`（`errors.go:12`）与标准库；`codec/json` 再经 build tag 选择外部库——依赖方向清晰（errors → codec/json 接口 → 具体实现），接口在消费方侧定义。
- **多二进制**：纯库，无独立二进制。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 错误聚合组件与类型位图 | [errors-architecture.html](errors-architecture.html) | architecture | showcase |
| 错误收集到序列化数据流图 | [errors-dataflow.html](errors-dataflow.html) | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 workflow/sequence/lifecycle 图——错误聚合是"写入→过滤→序列化"的数据加工管道（已由 dataflow 表达），无多角色泳道流程、无单次多方消息时序、也无单一实体状态机（错误只是被动数据载体），按资源节省原则省略。
