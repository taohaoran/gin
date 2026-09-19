# JSON 后端抽象（codec-json）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 域职责

本域为整个 gin 提供**统一的 JSON 编解码抽象**：对外只暴露 `codec/json.API`（`Core` 接口），对内用 Go build tag 在编译期从 encoding/json、sonic、jsoniter、go-json 四个后端中选定一个并在 `init()` 赋值给 `API`。
核心代码路径：使用方 `json.API.Marshal/Unmarshal/NewDecoder`（`binding/json.go`、`render/json.go`）→ 运行期实际落到被编译的那个后端实现。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|-----------|
| json-backend-abstraction | [json-backend-abstraction.md](json-backend-abstraction/json-backend-abstraction.md) | [架构图](json-backend-abstraction/json-backend-abstraction-architecture.html) | Core 接口 + 四后端 build tag 编译期分派 + init 赋值 |

## 3. 域级机制细节

- **编译期而非运行期分派**：同一时刻只编译一个后端，运行期零开销；构建时改 build tag 即可换后端。
- **语义对齐**：sonic 用 `ConfigStd`、jsoniter 用 `ConfigCompatibleWithStandardLibrary`，保证与标准库行为一致。
- **已知差异**：sonic/go-json 不传播 `http.MaxBytesError`，413 状态码在这些后端下可能落到默认 400（见 `context.go:838`）。
- **服务面**：同时被请求绑定（解码）与响应渲染（编码）两侧复用，是 gin 性能可插拔的关键底座。

## 4. 域级图

本域仅一个叶子，域级架构图即叶子 `json-backend-abstraction-architecture.html`，已展示 build tag 三后端分派与 `init → API` 主路径。
