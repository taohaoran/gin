# 数据绑定（binding）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 域职责

`binding` 域负责**把 HTTP 请求中的数据反序列化并校验到用户 struct**：覆盖请求体（JSON/XML/YAML/TOML/MsgPack/Protobuf/BSON/Plain）、URL 查询参数、表单（urlencoded/multipart）、路由 URI 参数与请求头五类来源。核心是 `Binding` 接口族 + 一套基于反射与 struct tag 的通用映射引擎 + 可插拔 struct 校验器。绑定结果失败时把错误交回 `Context`，由上层以 render 域渲染 400/413 错误响应。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| binding-core | [binding-core.md](binding-core/binding-core.md) | [架构图](binding-core/binding-core-architecture.html) | [时序图](binding-core/binding-core-sequence.html) | 接口族、反射映射引擎、默认校验器 |
| binding-formats | [binding-formats.md](binding-formats/binding-formats.md) | [架构图](binding-formats/binding-formats-architecture.html) | [数据流图](binding-formats/binding-formats-dataflow.html) | 14 种具体格式的 Binding 实现 |

## 3. 域级机制细节

- **三套接口分层**：`Binding`（从 *http.Request）、`BindingBody`（从字节，支持 body 复用）、`BindingUri`（从 map，不走请求体）。
- **统一 `validate`**：所有格式解码成功后都走 `validate(obj)` → `StructValidator`；仅 protobuf/bson 因生成结构无法加 tag 而跳过。
- **`binding.Default(method, contentType)`**：GET 强制 Form，其余按 MIME switch 选实例，default 兜底 Form。
- **build tag `nomsgpack`** 裁剪 msgpack 重依赖。
- **与 render 域衔接**：`Context.MustBindWith` 把绑定错误映射为 400/413 并 `Abort`，业务随后用 render 渲染错误体。

## 4. 域级图

![binding 域架构图](binding-architecture.html)

![请求绑定端到端时序](binding-sequence.html)

![binding 域输入渠道数据流](binding-dataflow.html)
