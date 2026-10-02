# 内部工具与可插拔 JSON codec（internal-utils）

> 本文是 `support` 域下的叶子子系统文档。域级总览见 [`../support.md`](../support.md)。
> 本文收拢 gin 散落各处的支撑代码：零拷贝字节转换 `internal/bytesconv`、路径文件系统 `internal/fs`、
> 顶层 `utils.go` 辅助函数、可插拔 JSON codec `codec/json` 四实现矩阵，以及 `deprecated.go`。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 零拷贝 string↔[]byte | `StringToBytes/BytesToString` 用 `unsafe` 重解释内存，零分配 | `internal/bytesconv/bytesconv.go:13`、`:19` |
| fs.FS 适配 | `internal/fs.FileSystem` 把 `http.FileSystem` 包成 `fs.FS` | `internal/fs/fs.go:9`、`:14` |
| 路由路径拼接 | `joinPaths` 保留相对路径尾斜杠语义 | `utils.go:136` |
| 监听地址解析 | `resolveAddress` 支持 `PORT` 环境变量、缺省 `:8080` | `utils.go:148` |
| 断言 | `assert1(guard, text)` 失败 panic（启动期编程错误防护） | `utils.go:86` |
| 中间件包装器 | `WrapF/WrapH` 把 `http.HandlerFunc/http.Handler` 包成 gin HandlerFunc | `utils.go:48`、`utils.go:55` |
| 绑定中间件 | `Bind(val)` 反射构造并绑定结构体，结果存入 `BindKey` | `utils.go:30` |
| 快捷 map 类型 | `H map[string]any`，自带 `MarshalXML` | `utils.go:62`、`:65` |
| Accept 头解析 | `parseAccept/filterFlags/chooseData` 内容协商辅助 | `utils.go:111`、`:92`、`:101` |
| 安全数值截断 | `safeInt8/safeUint16` 溢出封顶；`isASCII` 判定 | `utils.go:175`、`:183`、`:165` |
| JSON codec 接口 | `Core` 接口：Marshal/Unmarshal/MarshalIndent/NewEncoder/NewDecoder；`var API Core` 全局可换 | `codec/json/api.go:13`、`:10` |
| 四后端 build tag 实现 | 标准库默认 / goccy go-json / jsoniter / bytedance sonic（平台限定） | `codec/json/json.go`、`go_json.go`、`jsoniter.go`、`sonic.go` |
| 废弃 API | `Context.BindWith` 打印废弃日志并转调 MustBindWith | `deprecated.go:17` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `bytesconv.StringToBytes` | `internal/bytesconv/bytesconv.go:13` | `unsafe.Slice(unsafe.StringData(s), len(s))`——string 当 []byte 用，零拷贝；只读语义 |
| `fs.FileSystem` | `internal/fs/fs.go:9` | 内嵌 `http.FileSystem`，`Open` 返回 `fs.File` |
| `json.Core` 接口 | `codec/json/api.go:13` | 抽象 JSON 编解码；`Encoder/Decoder` 子接口（`api.go:22`、`:41`）暴露流式接口 |
| `json.API` 包变量 | `codec/json/api.go:10` | 编译期由某一实现的 `init()` 赋值，运行期全局唯一 |
| `H` | `utils.go:62` | `map[string]any` 快捷别名，业务侧最常用的响应载体 |
| `BindKey` | `utils.go:20` | 绑定中间件在 `c.Keys` 中存结构体实例的键 |

## 3. 关键调用链

### 3.1 可插拔 JSON codec 如何在编译期选定

1. `codec/json/api.go` 只声明 `var API Core` 与接口（`api.go:10`），不含实现。
2. 四个实现文件各自带 `//go:build` 约束，**同一构建只有一个生效**：
   - `json.go`（`json.go:5`）：`!jsoniter && !go_json && !(sonic && (linux||windows||darwin))`——缺省兜底，用标准库 `encoding/json`，`Package="encoding/json"`。
   - `go_json.go`（`go_json.go:5`）：tag `go_json` 时用 `github.com/goccy/go-json`。
   - `jsoniter.go`（`jsoniter.go:5`）：tag `jsoniter` 时用 `json-iterator/go`，配置 `ConfigCompatibleWithStandardLibrary`（`jsoniter.go:22`）。
   - `sonic.go`（`sonic.go:5`）：tag `sonic && (linux||windows||darwin)` 时用 bytedance/sonic，配置 `ConfigStd`（`sonic.go:22`）。
3. 每个生效实现的 `init()`（如 `json.go:17`）把自身单例赋值给 `json.API`。业务代码（如 errors.go `MarshalJSON` `errors.go:78`、render/json.go）统一调 `json.API.Marshal`，**对后端无感知**。

### 3.2 零拷贝转换的使用点

1. `bytesconv.StringToBytes(s)`（`bytesconv.go:13`）返回与原 string 共享底层字节的 `[]byte`，不分配；用于 `auth.go:37` 的 `subtle.ConstantTimeCompare` 与 `auth.go:93` 的 base64 编码。
2. `BytesToString(b)`（`bytesconv.go:19`）反向，用于 `recovery.go:100` 把 `httputil.DumpRequest` 的 `[]byte` 转回 string 做行处理。
3. **只读契约**：转换后的 slice/string 不得修改底层字节（unsafe 重解释），否则会破坏 Go string 不可变不变量。

### 3.3 监听地址与路径拼接

1. `resolveAddress(addr)`（`utils.go:148`）：0 个参数时读 `PORT` 环境变量（`utils.go:151`），未设置则 `:8080`；1 个参数直接用；多个参数 panic "too many parameters"（`utils.go:160`）。
2. `joinPaths(abs, rel)`（`utils.go:136`）：`path.Join` 后若相对路径以 `/` 结尾而结果不以 `/` 结尾，则补 `/`——保证路由组 basePath 的尾斜杠语义不丢（被 `routergroup.go:261` 调用）。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|---|---|---|
| JSON 后端选择 | 编译 tag：`-tags sonic` / `jsoniter` / `go_json`；缺省标准库 | `codec/json/*.go` build constraints |
| `PORT` 环境变量 | `resolveAddress` 无参数时读取，决定监听端口 | `utils.go:151` |
| sonic 平台限制 | 仅 linux/windows/darwin（依赖汇编加速） | `sonic.go:5` |
| `BindKey` 常量 | `_gin-gonic/gin/bindkey` | `utils.go:20` |

## 5. 错误与重试语义

- `assert1`（`utils.go:86`）失败即 panic：用在注册期/配置期编程错误（空凭证列表 `auth.go:78`、路径不合法 `routergroup.go:253`），不重试。
- `parseAccept/filterFlags` 是纯字符串解析，不返回错误；`chooseData`（`utils.go:101`）custom/wildcard 都为 nil 时 panic "negotiation config is invalid"。
- `bytesconv` 零拷贝无错误路径；`internal/fs.Open` 透传底层 `http.FileSystem` 的 error。
- JSON codec 的 `Marshal/Unmarshal` error 透传给调用方（如 errors.go、render/json），本叶子不处理。

## 6. 并发细节

- `json.API` 是包级可变变量，但**只在 `init()` 阶段赋值一次**，之后只读；请求期并发调用 `Marshal` 等方法——各 JSON 库实现自身并发安全（标准库/jsoniter/sonic 均为无状态函数式 API）。
- `bytesconv` 无状态纯函数，并发安全；但产出的 slice 与 string 共享内存，调用方不得并发写。
- `H`/`Params` 等是请求内数据结构，随 Context 池化。
- 无 goroutine/channel；`resolveAddress` 在 `Run` 启动期同步读环境变量。
- **internal 边界**：`internal/bytesconv`、`internal/fs` 只能被 `github.com/gin-gonic/gin/...` 内包 import，外部用户无法直接依赖——这是 Go internal 机制的天然隔离，当前被 recovery/auth/auth 等内部文件正常使用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `internal/bytesconv/bytesconv.go`、`internal/fs/fs.go`。
- `utils.go` 全部辅助函数与 `H` 类型。
- `codec/json/`：api.go 接口 + 四个 build-tag 实现。
- `deprecated.go`：`Context.BindWith` 废弃包装。

**Out-of-Scope（不在本仓库源码内）**
- 四个 JSON 后端库本体：`encoding/json`（标准库）、`github.com/goccy/go-json`、`github.com/json-iterator/go`、`github.com/bytedance/sonic`——均为外部依赖，本仓库只做适配层。
- `unsafe`/`os`/`path`/`reflect`/`runtime` 为标准库。
- binding/render 包如何调用 `json.API` 见各自域。

## 8. 与相邻子系统交互

- **被谁用**：recovery/auth 用 `bytesconv`；routergroup 用 `joinPaths`；errors 与 render/json 用 `json.API`；`Engine.Run` 用 `resolveAddress`。
- **依赖方向**：本叶子是**被依赖的底层**，不反向依赖中间件/路由/业务；`codec/json` 接口在本包定义，实现经 build tag 注入——依赖倒置（业务面向 `Core` 接口，不面向具体库）。
- **横向**：`utils.go` 的 `WrapF/WrapH` 把标准库 handler 桥接成 gin 中间件，是生态兼容缝。

## 9. 语言专项适配口径（Go）

- **internal 边界与依赖方向**：`internal/` 两个子包严格隔离，仅仓库内可用；`codec/json` 用"接口 + build tag 多实现"是 Go 经典可插拔模式——编译期静态选择，无运行时反射开销，符合 Go "小接口、显式装配" 风格。
- **unsafe 零拷贝**：`bytesconv` 用 Go 1.20+ 的 `unsafe.StringData/SliceData/String/Slice` 正规 API（而非旧式 `*(*string)(unsafe.Pointer)`），是官方认可的零拷贝惯用法。
- **build tags 多后端矩阵**：sonic 限定 linux/windows/darwin（依赖汇编 JIT），缺省标准库兜底——这是 Go CGO/平台特定库的常见分发策略。
- **并发模型**：无 goroutine/workqueue，纯函数式工具；`json.API` 只读单例天然并发安全。
- **多二进制**：纯库，无独立二进制。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 工具库与 JSON codec 多后端架构图 | [internal-utils-architecture.html](internal-utils-architecture.html) | architecture | showcase |
| JSON 编解码数据流图 | [internal-utils-dataflow.html](internal-utils-dataflow.html) | dataflow | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 sequence/workflow/lifecycle 图——工具函数是无状态纯调用（已由 architecture 拓扑 + dataflow 管道表达），无单次多方消息时序、无多角色泳道、也无单一实体状态机，按资源节省原则省略。
