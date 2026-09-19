# 请求绑定（request-binding）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 域职责

本域负责把 **HTTP 请求中的数据**（body、query、uri 路由参数、header、form 表单）绑定到用户的 Go struct，并完成结构校验。
核心代码路径：`Context.ShouldBind/MustBindWith`（`context.go`）→ `binding.Default(method, contentType)` 按 MIME 分派 → 各 `XxxBinding.Bind/BindBody/BindUri` 解码 → 统一 `validate(obj)` 经 `StructValidator` 校验 → error 经 Context 转 HTTP 400/413。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 职责一句话 |
|------|------|--------|-----------|
| binding-registry | [binding-registry.md](binding-registry/binding-registry.md) | [架构图](binding-registry/binding-registry-architecture.html) | Binding/BindingBody/BindingUri/StructValidator 接口、`Default` 按 MIME 分派、validator 包装 |
| body-bindings | [body-bindings.md](body-bindings/body-bindings.md) | [架构图](body-bindings/body-bindings-architecture.html) | JSON/XML/YAML/TOML/Protobuf/MsgPack/BSON/Plain 各 body 绑定器解码差异 |
| form-bindings | [form-bindings.md](form-bindings/form-bindings.md) | [架构图](form-bindings/form-bindings-architecture.html) | Form/Query/Uri/Header 绑定与 `mapFormByTag` 反射映射、multipart 文件上传 |

## 3. 域级机制细节

- **分派统一在 `Default`**：GET 一律 Form；其余按 Content-Type switch，default 落 Form。
- **校验统一收口**：各绑定器解码完成后都调包内 `validate(obj)`，`Validator` 置 nil 可整体关闭校验。
- **body 复用**：`ShouldBindBodyWith` 把读入的 body 缓存到 `Context` 的 `BodyBytesKey`，支持同一 body 多次绑定。
- **类型转换双轨**：form 系走反射 `setWithProperType`，body 系走各格式库 decoder，二者末尾汇合到 `validate`。

## 4. 域级图

本域不单独绘制域级架构图；三张叶子架构图已分别覆盖"分派注册表 / body 解码族 / form 反射映射"三条主路径。
