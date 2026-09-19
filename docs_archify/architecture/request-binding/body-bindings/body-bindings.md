# Body 绑定器族（body-bindings）

> 本文是 `request-binding/` 域下的叶子子系统文档。域级总览见 `../request-binding.md`。
> 本文展开各**请求体（body）绑定器**的解码实现差异；分派入口与校验接口见 `../binding-registry/`，
> Form/Query/Uri/Header 等非 body 绑定见 `../form-bindings/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| JSON body 绑定 | `jsonBinding`：`Bind` 读 `req.Body`，`BindBody` 读 `[]byte`，经 `codec/json` API 解码后 `validate` | `binding/json.go:27`、`binding/json.go:33`、`binding/json.go:44` |
| JSON 解码开关 | 包级 `EnableDecoderUseNumber`、`EnableDecoderDisallowUnknownFields` 控制 decoder 行为 | `binding/json.go:19`、`binding/json.go:25` |
| XML body 绑定 | `xmlBinding`：标准库 `encoding/xml` 解码 | `binding/xml.go:14`、`binding/xml.go:28` |
| YAML body 绑定 | `yamlBinding`：`goccy/go-yaml` 解码 | `binding/yaml.go:15`、`binding/yaml.go:29` |
| TOML body 绑定 | `tomlBinding`：`pelletier/go-toml/v2` 解码 | `binding/toml.go:15`、`binding/toml.go:29` |
| Protobuf body 绑定 | `protobufBinding`：先 `io.ReadAll`，类型断言为 `proto.Message` 后 `proto.Unmarshal`，**跳过 validate** | `binding/protobuf.go:15`、`binding/protobuf.go:29` |
| MsgPack body 绑定 | `msgpackBinding`：`ugorji/go/codec` 的 `MsgpackHandle`（受 `!nomsgpack` build tag 控制） | `binding/msgpack.go:17`、`binding/msgpack.go:31` |
| BSON body 绑定 | `bsonBinding`：先 `io.ReadAll` 再 `bson.Unmarshal`，不做 validate | `binding/bson.go:14`、`binding/bson.go:28` |
| Plain 文本绑定 | `plainBinding`：把 body 反射写入 `string` 或 `[]byte` 字段，其余类型报错 | `binding/plain.go:12`、`binding/plain.go:31` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `jsonBinding` | `binding/json.go:27` | 实现 `BindingBody`；`decodeJSON` 统一处理 decoder 开关 |
| `decodeJSON` | `binding/json.go:44` | 构造 decoder、按需 `UseNumber()`/`DisallowUnknownFields()`、`Decode` 后 `validate` |
| `xmlBinding`/`yamlBinding`/`tomlBinding` | `binding/{xml,yaml,toml}.go` | 结构同构：`Bind→decodeXxx(io.Reader)`，`BindBody→bytes.NewReader` |
| `protobufBinding` | `binding/protobuf.go:15` | `Bind` 内部 ReadAll 后转调 `BindBody`；`BindBody` 做 `proto.Message` 断言 |
| `msgpackBinding` | `binding/msgpack.go:17` | 用 `codec.NewDecoder(r, cdc).Decode(&obj)`，注意传入 `&obj` |
| `bsonBinding` | `binding/bson.go:14` | `Bind` ReadAll→`BindBody`→`bson.Unmarshal(body, obj)` |
| `plainBinding` | `binding/plain.go:12` | `decodePlain` 循环解指针后按 string/`[]byte` 反射设值 |

## 3. 关键调用链

1. **JSON body 绑定主路径**：`Context.ShouldBindJSON` → `b.Bind(c.Request, obj)`（`binding/json.go:33`）→ `decodeJSON(req.Body, obj)`（`binding/json.go:44`）→ `json.API.NewDecoder(r)` → 按开关调 `UseNumber()`/`DisallowUnknownFields()` → `decoder.Decode(obj)` → `validate(obj)`（`binding/json.go:55`）。
2. **body 复用绑定**：`Context.ShouldBindBodyWith`（`context.go:951`）先 `io.ReadAll(c.Request.Body)` 并 `c.Set(BodyBytesKey, body)` 缓存，再 `bb.BindBody(body, obj)`；各绑定器 `BindBody` 用 `bytes.NewReader(body)` 包成 reader 复用同一 `decodeXxx`。
3. **Protobuf 特例**：`Bind` → `io.ReadAll(req.Body)`（`binding/protobuf.go:22`）→ `BindBody` → `obj.(proto.Message)` 断言（`binding/protobuf.go:30`）→ `proto.Unmarshal`；源码注释明确"不能给 gen-proto 生成的 struct 加 `binding:""`，故返回 nil 跳过校验"（`binding/protobuf.go:37-40`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| `EnableDecoderUseNumber` | 默认 false；true 时数字解码为 `json.Number` 而非 `float64` | `binding/json.go:19` |
| `EnableDecoderDisallowUnknownFields` | 默认 false；true 时未知字段报错 | `binding/json.go:25` |
| `nomsgpack` build tag | 定义后不编译 msgpack 绑定器与 render | `binding/binding.go:5`、`binding/msgpack.go:5` |
| JSON 解码后端 | 实际由 `codec/json.API` 指向（encoding/json / sonic / jsoniter / go-json），见 `codec-json` 叶子 | `binding/json.go:13` |

## 5. 错误与重试语义

- 解码错误直接透传：`decoder.Decode` / `proto.Unmarshal` / `bson.Unmarshal` 的 error 原样返回，无重试。
- `jsonBinding.Bind` 对 `req==nil || req.Body==nil` 显式返回 `errors.New("invalid request")`（`binding/json.go:34`）。
- `protobufBinding.BindBody` 类型断言失败返回 `"obj is not ProtoMessage"`（`binding/protobuf.go:32`）。
- `plainBinding.decodePlain` 对非 string/`[]byte` 目标类型返回 `fmt.Errorf("type (%T) unknown type", v)`（`binding/plain.go:55`）。
- 错误经 Context 层转 HTTP 状态码：`MaxBytesError→413`，其余→400（见 `binding-registry` 第 5 节）。

## 6. 并发细节

- 各 `xxxBinding{}` 是无状态空结构体，并发安全，可在多 goroutine 间共享。
- `decodeJSON` 每次请求新建 decoder，无共享可变状态；`EnableDecoderUseNumber` 等是包级 bool，读多于写，框架启动期设置后只读。
- 本叶子无 goroutine/channel；`io.ReadAll`（protobuf/bson）是同步阻塞读。
- JSON 解码后端 `codec/json.API` 的选择发生在编译期（build tag），运行期为单一全局变量，并发读安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `binding/json.go`、`xml.go`、`yaml.go`、`toml.go`、`protobuf.go`、`msgpack.go`、`bson.go`、`plain.go`。

**Out-of-Scope（不在本仓库源码内）**
- `google.golang.org/protobuf/proto`、`github.com/ugorji/go/codec`、`go.mongodb.org/mongo-driver/v2/bson`、`github.com/goccy/go-yaml`、`github.com/pelletier/go-toml/v2`：第三方编解码库，**不在本仓库源码内**。
- JSON 后端抽象（`codec/json`）见 `codec-json` 叶子。
- 结构校验引擎见 `binding-registry` 叶子。

## 8. 与相邻子系统交互

- 上游：Context 的 `ShouldBindBodyWith/ShouldBindJSON` 等 → 调 `Bind`/`BindBody`。
- 本叶子 → 下游：JSON 绑定器依赖 `codec/json.API`（`binding/json.go:13`）；所有绑定器末尾依赖 `binding-registry` 的 `validate`。
- 平行：`form-bindings` 处理非 body 来源（form/query/uri/header）。

## 9. 语言专项适配口径（Go）

- **空结构体实现接口**：`jsonBinding{}` 等零值类型即满足 `BindingBody`，注册表中以包级 var 直接持有（`binding-registry`）。
- **build tag 条件编译**：msgpack 用 `//go:build !nomsgpack`，与 `codec/json` 的多后端 build tag 机制同属编译期裁剪，是 Go 库控制依赖体积的常用手法。
- **接口断言与安全降级**：protobuf 用 `obj.(proto.Message)` 安全断言（comma-ok），失败返回明确 error 而非 panic。
- 无 K8s 控制器模式、无 internal 越界。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Body 绑定器族架构图 | `body-bindings-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/body-bindings-architecture.json`。本叶子不补 sequence：各绑定器解码路径同构（decode→validate），主路径已在第 3 节文字化。
