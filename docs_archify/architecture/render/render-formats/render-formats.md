# 渲染格式实现（render-formats）

> 本文是 `render` 域下的叶子子系统文档。域级总览见 `../render.md`，本文逐个展开**各 Render 实现的内容类型与输出行为**；
> 共享的 `Render` 接口与写响应协议见 `../render-core/render-core.md`。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 渲染器 | Content-Type | 输出行为 | 源码路径 |
|--------|--------------|----------|----------|
| JSON | `application/json; charset=utf-8` | `codec/json.API.Marshal` 后写 | `render/json.go:19`、`json.go:67` |
| IndentedJSON | 同上 | `MarshalIndent` 4 空格缩进（仅开发用） | `render/json.go:24`、`json.go:78` |
| SecureJSON | 同上 | 数组结果前加 `Prefix`（防 JSON 劫持） | `render/json.go:29`、`json.go:94` |
| JsonpJSON | `application/javascript; charset=utf-8` | `callback(data);`，callback 经 JSEscape | `render/json.go:35`、`json.go:117` |
| AsciiJSON | `application/json` | 非 ASCII 字符转 `\uXXXX` | `render/json.go:41`、`json.go:155` |
| PureJSON | `application/json; charset=utf-8` | 流式 Encoder 且关闭 HTML 转义 | `render/json.go:46`、`json.go:184` |
| XML | `application/xml; charset=utf-8` | `encoding/xml` 流式 Encode | `render/xml.go:13`、`xml.go:20` |
| YAML | `application/yaml; charset=utf-8` | `goccy/go-yaml` Marshal | `render/yaml.go:14`、`yaml.go:21` |
| TOML | `application/toml; charset=utf-8` | `pelletier/go-toml/v2` Marshal | `render/toml.go:14`、`toml.go:21` |
| MsgPack | `application/msgpack; charset=utf-8` | `ugorji/go/codec` 编码；nomsgpack 裁剪 | `render/msgpack.go:22`、`msgpack.go:39` |
| ProtoBuf | `application/x-protobuf` | `proto.Marshal`（断言 `proto.Message`） | `render/protobuf.go:14`、`protobuf.go:21` |
| BSON | `application/bson` | `mongo-driver/v2/bson` Marshal | `render/bson.go:14`、`bson.go:21` |
| String | `text/plain; charset=utf-8` | `fmt.Fprintf` 格式化或纯写 format | `render/text.go:15`、`text.go:33` |
| HTML | `text/html; charset=utf-8` | `html/template` Execute/ExecuteTemplate | `render/html.go:46`、`html.go:92` |
| Data | 自定义 | 写 bytes 并设 `Content-Length` | `render/data.go:13`、`data.go:19` |
| Reader | 自定义 | `io.Copy` 流拷贝，支持自定义 headers | `render/reader.go:14`、`reader.go:22` |
| PDF | `application/pdf` | 直接写 PDF 二进制 | `render/pdf.go:10`、`pdf.go:17` |
| Redirect | 无（空实现） | `http.Redirect`，非法 code panic | `render/redirect.go:13`、`redirect.go:20` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|------|------|------|
| `JSON` 家族（6 变体） | `render/json.go:19-48` | 同一 `Data any`，差异化序列化/转义/前缀 |
| `HTMLProduction`/`HTMLDebug`/`HTML` | `render/html.go:30/36/46` | 生产复用模板 vs 调试每次重读；`Delims` 自定义分隔符 |
| `Reader` | `render/reader.go:14` | 通用流式响应，带 ContentLength 与额外 headers |
| `Redirect` | `render/redirect.go:13` | 携带 Code/Request/Location |

## 3. 关键调用链

### 链一：JSON 渲染
1. `Context.JSON` → `c.Render(code, render.JSON{Data: obj})`（`context.go:1252`）。
2. `JSON.Render`（`render/json.go:57`）→ `WriteJSON(w, r.Data)`（`json.go:67`）：先 `writeContentType`，再 `json.API.Marshal(obj)`，最后 `w.Write(jsonBytes)`。

### 链二：HTML 模板渲染
1. `Context.HTML`（`context.go:1221`）经 `engine.HTMLRender.Instance` 产出 `HTML`。
2. `HTML.Render`（`render/html.go:92`）：`Template == nil` 返 `errHTMLRendererNotConfigured`；`Name==""` 走 `Execute`，否则 `ExecuteTemplate`。
3. `HTMLDebug.loadTemplate`（`render/html.go:74`）每次按 Files/Glob/FileSystem 三种来源重新 `template.Must(...Parse...)`，未配置来源则 panic。

### 链三：Redirect
1. `Redirect.Render`（`render/redirect.go:20`）校验 code 必须在 3xx 或 201，否则 `panic`；随后 `http.Redirect(w, r.Request, r.Location, r.Code)`。其 `WriteContentType` 为空实现（`redirect.go:29`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|-------------|------|
| `engine.secureJSONPrefix` | SecureJSON 数组前缀 | `context.go:1243` |
| `Delims.Left/Right` | HTML 模板分隔符，默认 `{{`/`}}` | `render/html.go:16` |
| `Reader.ContentLength` | ≥0 时自动写 `Content-Length` | `render/reader.go:24` |

## 5. 错误与重试语义

- 序列化错误（如 `Marshal` 失败）直接返回，由 `Context.Render` 推入 `c.Errors` 并 Abort。
- **Redirect 用 panic** 而非 error 报告非法状态码（`redirect.go:22`）。
- HTML 未配置 renderer 返回错误而非 panic（`html.go:95`）；但 `HTMLDebug.loadTemplate` 无来源时 panic（`html.go:88`）。

## 6. 并发细节

- 全部为值类型无状态渲染，同步写 ResponseWriter；无 goroutine。
- `HTMLProduction.Template` 是并发只读共享的 `*template.Template`（生产期解析一次）；`HTMLDebug` 每次请求重新解析模板，便于开发热更新但牺牲性能。

## 7. 系统边界

**In-Scope（本仓库源码内）**：`render/` 下 json/xml/yaml/toml/msgpack/protobuf/bson/text/html/redirect/data/reader/pdf。

**Out-of-Scope（不在本仓库源码内）**：`encoding/xml`、`goccy/go-yaml`、`go-toml/v2`、`ugorji/go/codec`、`protobuf/proto`、`mongo-driver/bson`、`html/template`、`net/http` 等第三方/标准库。

## 8. 与相邻子系统交互

- **上游**：`Context` 各输出快捷方法（`context.go:1236-1332`）构造本叶子渲染器 → `Context.Render`（render-core）。
- **下游**：各渲染器 → 第三方序列化库 / `html/template` / `http.Redirect` → `http.ResponseWriter`。

## 9. 语言专项适配口径（Go）

- **值类型接口实现**：每个渲染器是小 struct（`Data any`），实现 `Render` 接口；`render.go:17` 编译期断言。
- **流式 vs 缓冲**：`XML/PureJSON/Reader` 直接把 encoder/copy 接到 ResponseWriter（流式省内存）；多数 JSON 变体先 Marshal 到 `[]byte` 再写（便于前缀/转义后处理）。
- **无并发原语**：纯同步写。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| render-formats 架构图 | `render-formats-architecture.html` | architecture | showcase |
| 响应渲染数据流 | `render-formats-dataflow.html` | dataflow | showcase |

- JSON IR 源：`json/render-formats-architecture.json`、`json/render-formats-dataflow.json`。
- 省略 sequence/lifecycle：各渲染器是同构短函数，时序已由 render-core 的 `Context.Render` 覆盖；无单实体状态机，按资源节省原则省略。
