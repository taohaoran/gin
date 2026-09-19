# 错误模型（error-model）

> 本文是 `infra` 域下的叶子子系统文档。域级总览见 `../infra.md`，
> 本文只展开 `Context` 上的错误收集与类型体系，不重复中间件如何消费错误（见 builtin-middleware 域）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `errors.go`（174 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Error` 结构 | `{Err, Type, Meta}` 包装一个请求期错误 | `errors.go:32` |
| `ErrorType` 位标志 | 64 位错误类型码，可按位或组合 | `errors.go:16` |
| 类型常量 | `Bind/Render/Private/Public/Any` | `errors.go:18-29` |
| `errorMsgs` 错误链 | `[]*Error`，挂在 `c.Errors` | `errors.go:38` |
| `SetType/SetMeta` | 链式设置类型与元数据 | `errors.go:43`、`errors.go:49` |
| `IsType` | 位与判断 `(Type & flags) > 0` | `errors.go:87` |
| `Unwrap` | 兼容 `errors.Is/As` | `errors.go:92` |
| `JSON()/MarshalJSON` | 按 Meta 类型展开并走 codec/json 序列化 | `errors.go:55`、`errors.go:77` |
| `ByType` | 按位标志过滤错误链 | `errors.go:98` |
| `Last/Errors/String` | 取末个/扁平化/可读串 | `errors.go:116`、`errors.go:130`、`errors.go:161` |

对外暴露点：`c.Error(err)`（`context.go:262`）把任意 error 包装成 `Error{Type: ErrorTypePrivate}` 追加进 `c.Errors`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `ErrorType` | `errors.go:16` | `uint64` 别名；位标志而非枚举 |
| `Error` | `errors.go:32` | 实现 `error`/`Unwrap`/`json.Marshaler` |
| `errorMsgs` | `errors.go:38` | `[]*Error`，方法集 `ByType/Last/Errors/JSON/String` |
| `ErrorTypeAny` | `errors.go:28` | `1<<64 - 1`，全 1 位掩码，匹配所有类型 |

类型位分配（高位互斥领域位 + 低位可见性位）：
- `ErrorTypeBind = 1<<63`（绑定失败）、`ErrorTypeRender = 1<<62`（渲染失败）
- `ErrorTypePrivate = 1<<0`（私有，仅服务器日志）、`ErrorTypePublic = 1<<1`（可公开给客户端）

## 3. 关键调用链

**主路径：`c.Error(err)` 包装并追加（与 context.go 协作）**

1. 调用方 `c.Error(err)`（`context.go:262`）；nil 直接 panic（`context.go:263-265`）。
2. 用 `errors.As(err, &parsedError)` 判断是否已经是 `*Error`（`context.go:268`）；不是则包成 `&Error{Err: err, Type: ErrorTypePrivate}`（`context.go:270-273`）。
3. `c.Errors = append(c.Errors, parsedError)`（`context.go:276`），返回该 `*Error` 供链式 `SetType/SetMeta`。

**类型过滤主路径（`errorMsgs.ByType`，`errors.go:98`）**

1. 空切片直接返回 nil（`errors.go:99-101`）。
2. `typ == ErrorTypeAny` 原样返回全链（`errors.go:102-104`）。
3. 遍历每条 `msg.IsType(typ)`——即 `(msg.Type & flags) > 0`（`errors.go:87-88`），命中则收集进结果（`errors.go:106-110`）。

**JSON 序列化主路径（`Error.JSON()`，`errors.go:55`）**

1. `Meta` 为 nil 时只放 `{"error": msg.Error()}`（`errors.go:70-72`）。
2. `Meta` 为 struct 直接返回；为 map 逐键展开进 `H{}`；其它类型放进 `jsonData["meta"]`（`errors.go:58-68`）。
3. `Error.MarshalJSON` 调 `json.API.Marshal(msg.JSON())`（`errors.go:77-79`），走可插拔 JSON 后端（codec/json 叶子）。
4. `errorMsgs.MarshalJSON` 对 0/1/多条分别返回 nil / 单对象 / 数组（`errors.go:141-154`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `Error.Type` 默认值 | `c.Error` 包装时默认 `ErrorTypePrivate` | `context.go:272` |
| `ErrorTypeAny` | 全 1 掩码，`ByType(ErrorTypeAny)` 返回全部 | `errors.go:28`、`errors.go:102` |
| 可见性位 | Private 仅服务端日志；Public 可返回客户端；两者位不同可组合 | `errors.go:23-26` |

## 5. 错误与重试语义

- 本模型是**错误收集与分类**设施，不负责重试；错误经 `c.Error` 入链后由中间件/handler 决定如何处理。
- `Error.Unwrap` 返回 `msg.Err`（`errors.go:92-94`），使 `errors.Is/As` 能穿透到被包装的原始 error。
- `ErrorTypeBind/Render` 高位标记"发生在哪一阶段"，便于日志/响应按类型过滤。
- 序列化失败由 `json.API.Marshal` 返回 error，本层不吞。

## 6. 并发细节

- 不创建 goroutine。
- `c.Errors` 是 per-request 切片，随 `Context` 在 `sync.Pool` 中复用；`c.Error` 的 append 在请求 goroutine 内进行。
- `Context.mu sync.RWMutex` 保护 `Keys`，但 `Errors` 切片本身按请求串行访问（Gin 约定同一请求不并发写 `c.Errors`）。
- 类型位用 `uint64` 位运算，无锁、无原子需求（值语义）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `errors.go` 全部：错误结构、类型位、错误链方法、JSON 序列化。

**Out-of-Scope（不在本仓库源码内）**
- `reflect`（`errors.go:10`）：按 Meta 类型展开，标准库外部。
- `codec/json`（`errors.go:12`）：`json.API.Marshal` 可插拔后端，见 json-backend-abstraction 叶子。
- `Context.Error` 的包装逻辑在 `context.go`（context-object 叶子）。
- 本叶子不定义日志中间件如何消费 Private/Public（logger/recovery 叶子）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：所有 handler/绑定/渲染失败都经 `c.Error`（context-object 叶子）把 error 喂进 `c.Errors`。
- 本叶子 → 下游：`Logger` 中间件读 `c.Errors.ByType(ErrorTypePrivate)` 写日志（`logger.go:302`）；`ErrorLoggerT` 用 `c.JSON(-1, c.Errors.ByType(typ))` 响应客户端（`logger.go:215-217`）。
- 本叶子 → 相邻：序列化委托 `codec/json`（json-backend 叶子）。

## 9. 语言专项适配口径

- **错误包装链**：实现 `error` + `Unwrap()`，符合 Go 1.13+ `errors.Is/As/Unwrap` 协议；类型用位标志而非枚举，支持"一个错误同时是 Bind 又是 Public"这类组合。
- **JSON 抽象**：`MarshalJSON` 不直接 `encoding/json`，而是走 `json.API`，与 codec/json 后端切换解耦。
- 无 goroutine/channel；纯值类型数据结构。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 错误模型架构图 | `error-model-architecture.html` | architecture | showcase |
| JSON IR | `json/error-model-architecture.json` | — | — |

本叶子补 architecture 而非 sequence：重点是 Error/ErrorType/errorMsgs/JSON 后端的组件边界与类型体系，调用链已在第 3 节文字化。
