# JSON 后端抽象与编译期分派（json-backend-abstraction）

> 本文是 `codec-json/` 域下的唯一叶子子系统文档。域级总览见 `../codec-json.md`。
> 本文展开 `codec/json` 包如何用"统一接口 + build tag 编译期二选一"在运行期透明切换 JSON 编码后端。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 统一 API 变量 | 包级 `var API Core`，运行期指向选中后端，供 binding/render 统一调用 | `codec/json/api.go:10` |
| Core 接口 | `Marshal/Unmarshal/MarshalIndent/NewEncoder/NewDecoder` 五方法 | `codec/json/api.go:13` |
| Encoder/Decoder 子接口 | `SetEscapeHTML/Encode`、`UseNumber/DisallowUnknownFields/Decode` | `codec/json/api.go:22`、`codec/json/api.go:41` |
| 默认后端 encoding/json | build tag 为 `!jsoniter && !go_json && !(sonic && 三平台)` 时编译 | `codec/json/json.go:5`、`codec/json/json.go:17` |
| sonic 后端 | build tag `sonic && (linux|windows|darwin)`，字节跳动 sonic | `codec/json/sonic.go:5`、`codec/json/sonic.go:18` |
| jsoniter 后端 | build tag `jsoniter`，`ConfigCompatibleWithStandardLibrary` | `codec/json/jsoniter.go:5`、`codec/json/jsoniter.go:18` |
| go-json 后端 | build tag `go_json`，`goccy/go-json` | `codec/json/go_json.go:5`、`codec/json/go_json.go:18` |
| 后端标识常量 | 各后端 `const Package` 声明当前所用库 | `codec/json/json.go:15`、`sonic.go:16`、`jsoniter.go:16`、`go_json.go:16` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `API`（包级 var） | `codec/json/api.go:10` | 唯一对外入口，运行期被某后端 `init()` 赋值 |
| `Core` 接口 | `codec/json/api.go:13` | 抽象 JSON 编解码能力，屏蔽后端差异 |
| `Encoder`/`Decoder` 接口 | `codec/json/api.go:22`、`:41` | 流式编码器/解码器抽象，含 UseNumber 等开关 |
| `jsonApi`/`sonicApi`/`jsoniterApi`/`gojsonApi` | 各 `*.go` | 各后端的空结构体实现，方法一一转调对应库 |

## 3. 关键调用链

1. **统一调用**：绑定侧 `decodeJSON`（`binding/json.go:45`）用 `json.API.NewDecoder(r)`，渲染侧 `WriteJSON`（`render/json.go:69`）用 `json.API.Marshal(obj)`——两侧都不感知具体后端。
2. **编译期选后端**：构建时按 build constraint 只编译一个后端文件。例如默认后端 `json.go` 的约束 `!jsoniter && !go_json && !(sonic && (linux || windows || darwin))`（`codec/json/json.go:5`），即未显式指定任何后端时才用标准库。
3. **init 赋值**：被编译的后端在 `init()` 中执行 `API = jsonApi{}`（`codec/json/json.go:17-19`，`sonic.go:18-20`、`jsoniter.go:18-20`、`go_json.go:18-20` 同构）；sonic 内部用 `sonic.ConfigStd`、jsoniter 用 `ConfigCompatibleWithStandardLibrary` 保证与标准库语义兼容。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| build tag `jsoniter` | 启用 json-iterator/go 后端 | `codec/json/jsoniter.go:5` |
| build tag `go_json` | 启用 goccy/go-json 后端 | `codec/json/go_json.go:5` |
| build tag `sonic` | 启用 bytedance/sonic，限 linux/windows/darwin | `codec/json/sonic.go:5` |
| 无 build tag | 默认 encoding/json | `codec/json/json.go:5` |
| `EnableDecoderUseNumber`/`DisallowUnknownFields` | 由调用侧在 decoder 上调用，与后端无关 | `binding/json.go:19`、`binding/json.go:25` |

## 5. 错误与重试语义

- 无重试；Marshal/Unmarshal 错误直接透传各底层库。
- 代码兼容性风险：`Context.MustBindWith` 注释指出 sonic/go-json 不传播 `http.MaxBytesError`（`context.go:838-840`），故 413 状态码在这些后端下可能落到默认 400——这是后端抽象的已知语义差异。
- 各后端均选 `ConfigStd`/`ConfigCompatibleWithStandardLibrary`，保证 API 语义与标准库一致，切换后端不改业务代码。
- 构建时若同时指定互斥 tag，Go 编译器因约束互斥只选其一，不会出现两个 `init()` 重复赋值 `API`。

## 6. 并发细节

- `API` 是包级 var，在 `init()` 阶段赋值完成后运行期只读，并发读安全。
- 各 `xxxApi{}` 为空结构体，方法无共享可变状态；底层库（sonic 等）自身做了并发优化。
- 无 goroutine/channel/锁；分派是编译期行为，运行期零开销。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `codec/json/{api.go, json.go, sonic.go, jsoniter.go, go_json.go}`。

**Out-of-Scope（不在本仓库源码内）**
- `encoding/json`（标准库）、`github.com/bytedance/sonic`、`github.com/json-iterator/go`、`github.com/goccy/go-json`：真正的 JSON 实现，**不在本仓库源码内**。
- 使用方（`binding/json.go`、`render/json.go`）见对应叶子。

## 8. 与相邻子系统交互

- 上游（消费方）：`binding/json.go`（解码请求体）、`render/json.go`（编码响应）、`binding/form_mapping.go`（struct/map 字段二次 Unmarshal）。
- 本叶子 → 下游：四个第三方/标准库 JSON 库，编译期绑定其一。
- 价值：业务与渲染/绑定代码只依赖 `codec/json.API` 接口，切换高性能后端只改 build tag，不改业务代码。

## 9. 语言专项适配口径（Go）

- **编译期策略模式（build tag 分派）**：用 Go build constraints 替代运行期 if/else 工厂，零运行期开销、二进制只含一个后端——是 Go 库做"后端可插拔"的经典手法（对比 Java 的 SPI/反射）。
- **接口隔离**：`Core` 接口把流式 Encoder/Decoder 也抽象成接口（`Encoder`/`Decoder`），使 `SetEscapeHTML`/`UseNumber` 等标准库方法在所有后端下可用。
- **语义对齐配置**：sonic 用 `ConfigStd`、jsoniter 用 `ConfigCompatibleWithStandardLibrary`，避免切换后端导致行为漂移。
- 无 K8s 控制器模式；无 internal 越界。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| JSON 后端抽象与编译期分派架构图 | `json-backend-abstraction-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/json-backend-abstraction-architecture.json`。本叶子不补 sequence：分派发生在编译期、运行期只是接口调用，时序无独立价值。
