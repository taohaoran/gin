# 绑定格式实现（binding-formats）

> 本文是 `binding` 域下的叶子子系统文档。域级总览见 `../binding.md`，本文逐个展开**各具体格式的 Binding 实现**；
> 共享的接口族、反射映射引擎与校验器见 `../binding-core/binding-core.md`。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 实现 | 绑定类型 | 行为要点 | 源码路径 |
|------|----------|----------|----------|
| JSON | `BindingBody` | 经 `codec/json.API` 解码；支持 `EnableDecoderUseNumber`/`EnableDecoderDisallowUnknownFields` 开关 | `binding/json.go:27`、`json.go:44` |
| XML | `BindingBody` | `encoding/xml` 解码后校验 | `binding/xml.go:14`、`xml.go:28` |
| YAML | `BindingBody` | `goccy/go-yaml` 解码后校验 | `binding/yaml.go:15`、`yaml.go:29` |
| TOML | `BindingBody` | `pelletier/go-toml/v2` 解码后校验 | `binding/toml.go:15`、`toml.go:29` |
| MsgPack | `BindingBody` | `ugorji/go/codec` msgpack 解码；build tag `nomsgpack` 可裁剪 | `binding/msgpack.go:17`、`msgpack.go:31` |
| Protobuf | `BindingBody` | `io.ReadAll` 读 body 后 `proto.Unmarshal`；**故意跳过校验** | `binding/protobuf.go:21`、`protobuf.go:29` |
| BSON | `BindingBody` | `mongo-driver/v2/bson` 反序列化；不校验 | `binding/bson.go:20`、`bson.go:28` |
| Plain | `BindingBody` | 把 body 原样写入 `*string` 或 `*[]byte`，否则报错 | `binding/plain.go:12`、`plain.go:31` |
| Form | `Binding` | `ParseForm`+`ParseMultipartForm` 后用 `req.Form` 反射映射 | `binding/form.go:24` |
| FormPost | `Binding` | 仅 `ParseForm`，用 `req.PostForm`（不含 URL query） | `binding/form.go:41` |
| FormMultipart | `Binding` | `ParseMultipartForm` 后用 `multipartRequest` 映射（含文件） | `binding/form.go:55` |
| Query | `Binding` | `req.URL.Query()` 反射映射 | `binding/query.go:15` |
| Header | `Binding` | `headerSource` 把 key 规范化为 MIME 头后反射映射 | `binding/header.go:19` |
| Uri | `BindingUri` | `mapURI` 后校验（见 binding-core） | `binding/uri.go:13` |

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|------|------|------|
| `jsonBinding` | `binding/json.go:27` | JSON 绑定，持有两个包级开关 `EnableDecoderUseNumber`/`EnableDecoderDisallowUnknownFields` |
| `xmlBinding`/`yamlBinding`/`tomlBinding`/`msgpackBinding` | 各格式文件 | 空结构体，`Bind` 读 `req.Body`、`BindBody` 读字节，模式一致 |
| `protobufBinding` | `binding/protobuf.go:15` | 断言 `obj.(proto.Message)`，否则报错 |
| `bsonBinding` | `binding/bson.go:14` | BSON 反序列化 |
| `plainBinding` | `binding/plain.go:12` | 纯文本/字节绑定，仅支持 string/[]byte 目标 |
| `formBinding`/`formPostBinding`/`formMultipartBinding` | `binding/form.go:14` | 三种表单绑定，差别在解析哪份 map |
| `queryBinding`/`headerBinding` | `binding/query.go:9`、`header.go:13` | 参数/请求头绑定 |
| `defaultMemory` | `binding/form.go:12` | multipart 内存上限常量 `32 << 20`（32MB） |

## 3. 关键调用链

### 链一：JSON 绑定（Body 类代表）
1. `Context.ShouldBindJSON` → `ShouldBindWith(obj, binding.JSON)` → `jsonBinding.Bind(req, obj)`（`binding/json.go:33`）。
2. `Bind` 判空 `req.Body` 后调 `decodeJSON(req.Body, obj)`（`binding/json.go:44`）。
3. `decodeJSON` 用 `json.API.NewDecoder(r)` 创建解码器，按两个开关调用 `UseNumber()`/`DisallowUnknownFields()`，`Decode(obj)` 后 `validate(obj)`（`binding/json.go:45`-`json.go:55`）。

### 链二：Form 绑定（表单类代表）
1. `formBinding.Bind`（`binding/form.go:24`）先 `req.ParseForm()`，再 `req.ParseMultipartForm(defaultMemory)`，非 multipart 错误 `http.ErrNotMultipart` 被忽略。
2. 用合并后的 `req.Form` 调 `mapForm(obj, req.Form)` → 进入 binding-core 反射映射，末尾 `validate(obj)`（`binding/form.go:31`）。

### 链三：Default 分发
1. `binding.Default(method, contentType)`（`binding/binding.go:95`）：GET 一律返回 `Form`；否则按 MIME switch 选中对应实例（JSON/XML/ProtoBuf/MsgPack/YAML/TOML/FormMultipart/BSON），default 兜底 `Form`。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|-------------|------|
| `EnableDecoderUseNumber` | `false`；true 时 JSON 数字解码为 `json.Number` 而非 float64 | `binding/json.go:19` |
| `EnableDecoderDisallowUnknownFields` | `false`；true 时多余字段报错 | `binding/json.go:25` |
| `defaultMemory` | `32 << 20`（32MB），multipart 内存上限 | `binding/form.go:12` |
| build tag `nomsgpack` | 裁剪 msgpack 实现，`MsgPack` 实例消失（见 `binding_nomsgpack.go`） | `binding/msgpack.go:5` |

## 5. 错误与重试语义

- **无重试**：每个 `Bind` 一次性解码，错误直接返回。
- **JSON**：`req`/`req.Body` 为 nil 返回 `errors.New("invalid request")`（`json.go:35`）；解码错误透传。
- **Protobuf**：`obj` 非 `proto.Message` 返回 `errors.New("obj is not ProtoMessage")`（`protobuf.go:31`）。
- **Plain**：目标非 string/[]byte 返回 `fmt.Errorf("type (%T) unknown type")`（`plain.go:55`）。
- **Form**：`ParseMultipartForm` 仅容忍 `http.ErrNotMultipart`，其它错误（如超 `MaxBytesError`）上抛，由 `MustBindWith` 映射为 413。
- **BSON/Protobuf**：解码后不做 struct 校验，错误仅来自解码本身。

## 6. 并发细节

- 所有 binding 均为**无状态空结构体单例**，并发安全；不创建 goroutine。
- JSON 解码依赖 `codec/json.API` 包级接口，其具体实现（标准库/go-json/jsoniter/sonic）在初始化期选定，运行期只读共享。
- `io.ReadAll(req.Body)`（protobuf/bson/plain）会一次性读全 body，受 server 整体并发与内存约束。

## 7. 系统边界

**In-Scope（本仓库源码内）**：`binding/` 下 json/xml/yaml/toml/msgpack/protobuf/bson/plain/header/query/form 各格式文件。

**Out-of-Scope（不在本仓库源码内）**：
- `encoding/json`(经 codec/json)、`encoding/xml`、`goccy/go-yaml`、`pelletier/go-toml/v2`、`ugorji/go/codec`、`google.golang.org/protobuf/proto`、`mongo-driver/v2/bson` 等第三方编解码库。
- 反射映射与校验本身（binding-core）。

## 8. 与相邻子系统交互

- **上游**：`Context` 快捷方法（`ShouldBindJSON/ShouldBindXML/...`，`context.go:890` 等）→ 本叶子各 binding；`binding.Default`（binding-core）按 MIME 选择本叶子实例。
- **下游**：各 binding → 第三方解码器；表单类 → binding-core 反射映射；解码成功 → binding-core `validate`。

## 9. 语言专项适配口径（Go）

- **接口实现矩阵**：本叶子是 Go "小结构体实现接口"模式的密集样本——14 个空结构体分别满足 `Binding`/`BindingBody`/`BindingUri`，由 `binding.Default` 按 MIME 分发，是典型策略模式。
- **条件编译**：`//go:build !nomsgpack` 与 `nomsgpack` 两份文件（`binding.go`/`msgpack.go` vs `binding_nomsgpack.go`）用 build tag 裁剪可选重依赖，减小二进制体积。
- **无并发原语**：纯函数式解码，无 goroutine/锁；与 K8s 控制器模式无关。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| binding-formats 架构图 | `binding-formats-architecture.html` | architecture | showcase（首版 ctx→disp 间距不足落 standard，右移 disp 至 x=310 并缩短标签为"方法/类型"后通过） |
| 请求数据绑定管道数据流 | `binding-formats-dataflow.html` | dataflow | showcase（首版 viewBox 过窄且两条分发边标签重叠，加宽 viewBox 至 1200、"表单类"边走 bottom-channel 后通过） |

- JSON IR 源：`json/binding-formats-architecture.json`、`json/binding-formats-dataflow.json`。
- 省略 sequence/lifecycle：本叶子各格式实现是同构的短解码函数，时序与 binding-core 已给的反射链路重复；无单实体状态机，按资源节省原则省略。
