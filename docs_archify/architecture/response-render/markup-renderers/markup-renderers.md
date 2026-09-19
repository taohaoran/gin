# Markup 渲染器族（markup-renderers）

> 本文是 `response-render/` 域下的叶子子系统文档。域级总览见 `../response-render.md`。
> 本文展开 JSON 之外的 markup/二进制/文本渲染器；JSON 变体与 Render 接口见 `../render-interface/`，
> HTML/Redirect/Reader 见 `../html-redirect-reader/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| XML 渲染 | `XML{Data}`：`xml.NewEncoder(w).Encode` 直写，Content-Type `application/xml; charset=utf-8` | `render/xml.go:13`、`render/xml.go:20` |
| YAML 渲染 | `YAML{Data}`：`goccy/go-yaml` Marshal 后 w.Write | `render/yaml.go:14`、`render/yaml.go:21` |
| TOML 渲染 | `TOML{Data}`：`pelletier/go-toml/v2` Marshal 后 w.Write | `render/toml.go:14`、`render/toml.go:21` |
| ProtoBuf 渲染 | `ProtoBuf{Data}`：类型断言为 `proto.Message` 后 `proto.Marshal` | `render/protobuf.go:14`、`render/protobuf.go:21` |
| MsgPack 渲染 | `MsgPack{Data}`：`ugorji/go/codec` 的 `MsgpackHandle` 编码器（`!nomsgpack` build tag） | `render/msgpack.go:22`、`render/msgpack.go:39` |
| BSON 渲染 | `BSON{Data}`：`bson.Marshal(&r.Data)` | `render/bson.go:14`、`render/bson.go:21` |
| Data 二进制渲染 | `Data{ContentType, Data}`：自定 Content-Type，非空时写 `Content-Length` | `render/data.go:13`、`render/data.go:19` |
| String 文本渲染 | `String{Format, Data}`：`fmt.Fprintf` 格式化或直接写 format | `render/text.go:15`、`render/text.go:33` |
| PDF 渲染 | `PDF{Data}`：透传 PDF 二进制，Content-Type `application/pdf` | `render/pdf.go:10`、`render/pdf.go:17` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `XML`/`YAML`/`TOML`/`ProtoBuf`/`MsgPack`/`BSON` | 各 `render/*.go` | 持有 `Data any` 的值对象，实现 `Render` 接口 |
| `Data` | `render/data.go:13` | 持有 `ContentType` 与 `[]byte`，最通用的二进制响应 |
| `String` | `render/text.go:15` | 持有 `Format` 与 `[]any`，文本/格式化输出 |
| `PDF` | `render/pdf.go:10` | 纯 `[]byte` 透传，固定 Content-Type |
| `WriteMsgPack`/`WriteString` | `render/msgpack.go:39`、`render/text.go:33` | 可复用的包级写出函数 |

## 3. 关键调用链

1. **XML 渲染主路径**：`Context.XML(code, obj)`（`context.go:1278`）→ `c.Render(code, render.XML{Data:obj})` → `XML.Render`（`render/xml.go:20`）：先 `WriteContentType` → `xml.NewEncoder(w).Encode(r.Data)` 直接流式编码到 w。
2. **YAML/TOML/BSON 渲染**：先 Marshal 成 `[]byte` 再 `w.Write`（如 `render/yaml.go:24-30`、`render/bson.go:24-27`）；与 XML 的"直接流式 Encode"不同，多一次内存字节拷贝。
3. **Data 二进制渲染**：`Data.Render`（`render/data.go:19`）：`WriteContentType` → 若 `len(r.Data)>0` 写 `Content-Length` → `w.Write(r.Data)`；String 渲染 `len(data)>0` 走 `fmt.Fprintf`，否则直接写 format 串（`render/text.go:35-40`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| 各 Content-Type | XML/YAML/TOML/PB/MsgPack/BSON/PDF 各自固定常量 | 各 `render/*.go` |
| `Data.ContentType` | 用户自定，无默认 | `render/data.go:14` |
| `Content-Length` | Data 仅在 body 非空时写 | `render/data.go:21` |
| `nomsgpack` build tag | 定义后不编译 MsgPack 渲染器 | `render/msgpack.go:5` |

## 5. 错误与重试语义

- 无重试：Marshal/Encode/w.Write 出错即返回 error，由 `Context.Render` 统一 `c.Error+c.Abort`。
- `ProtoBuf.Render` 对 `r.Data.(proto.Message)` 直接类型断言（`render/protobuf.go:24`），非 Message 会 panic——调用方需保证传入 proto.Message。
- `BSON.Render` 用 `bson.Marshal(&r.Data)`，错误经 `if err == nil` 守卫后才写（`render/bson.go:25`）。
- 错误一旦发生在已写 header 之后，HTTP 状态码已提交，无法回退。

## 6. 并发细节

- 全部渲染器为无状态值对象，并发安全。
- XML 用 `xml.NewEncoder(w)` 流式编码，不额外占大内存；YAML/TOML/BSON 先全量 Marshal 到内存再写。
- 本叶子无 goroutine/channel/锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `render/{xml,yaml,toml,protobuf,msgpack,bson,data,text,pdf}.go`。

**Out-of-Scope（不在本仓库源码内）**
- `encoding/xml`（标准库）、`goccy/go-yaml`、`pelletier/go-toml/v2`、`google.golang.org/protobuf/proto`、`ugorji/go/codec`、`go.mongodb.org/mongo-driver/v2/bson`：第三方/标准库编码，**不在本仓库源码内**。
- `Render` 接口与 Content-Type 幂等写入见 `render-interface` 叶子。

## 8. 与相邻子系统交互

- 上游：`Context.XML/YAML/TOML/ProtoBuf/BSON/String/Data/PDF`（`context.go:1278` 起）。
- 本叶子 → 下游：`ResponseWriter`；编码依赖各第三方库。
- 与 `render-interface`、`html-redirect-reader` 共同实现 `Render` 接口。

## 9. 语言专项适配口径（Go）

- **值对象 + 方法集**：所有渲染器为 struct 值类型，方法用值接收者，符合 Go 接口实现惯例。
- **流式 vs 全量两种编码风格**：XML 直接 Encode 到 w（低内存），YAML/BSON 先 Marshal 到 []byte（简单但占内存），是 Go 生态常见取舍。
- **build tag 裁剪**：MsgPack 同绑定侧用 `!nomsgpack` 同步裁剪。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Markup 渲染器族架构图 | `markup-renderers-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/markup-renderers-architecture.json`。本叶子不补 sequence：各渲染路径同构（Marshal/Encode→写 w）。
