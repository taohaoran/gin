# 绑定注册表与校验器（binding-registry）

> 本文是 `request-binding/` 域下的叶子子系统文档。域级总览见 `../request-binding.md`。
> 本文只展开绑定器接口契约、MIME 分派注册表与结构校验器包装层；各具体 Body 绑定器实现见 `../body-bindings/`，
> Form/Query/Uri/Header 映射实现见 `../form-bindings/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 绑定器接口族 | `Binding`（`Name`+`Bind`）、`BindingBody`（追加 `BindBody([]byte)`）、`BindingUri`（追加 `BindUri(map)`） | `binding/binding.go:32`、`binding/binding.go:39`、`binding/binding.go:46` |
| 结构校验接口 | `StructValidator`（`ValidateStruct`+`Engine()`），供框架整体替换校验引擎 | `binding/binding.go:55` |
| MIME 常量集 | `MIMEJSON`/`MIMEXML`/`MIMEPOSTForm`/`MIMEMultipartPOSTForm`/`MIMEPROTOBUF`/`MIMEMSGPACK`/`MIMEYAML`/`MIMETOML`/`MIMEBSON` 等 | `binding/binding.go:12` |
| 全局绑定注册表 | 包级变量 `JSON/XML/Form/Query/FormPost/FormMultipart/ProtoBuf/MsgPack/YAML/Uri/Header/Plain/TOML/BSON` 共 14 个 | `binding/binding.go:76` |
| 按 MIME 分派 | `Default(method, contentType)`：GET 一律 Form；其余按 Content-Type switch 分派，default 落到 Form | `binding/binding.go:95` |
| 默认校验器 | `defaultValidator` 包装 `go-playground/validator/v10`，`sync.Once` 懒初始化，tag 名固定为 `binding` | `binding/default_validator.go:16`、`binding/default_validator.go:93` |
| 校验类型分流 | `ValidateStruct` 按 reflect Kind 分流：Ptr→解引用、Struct→校验、Slice/Array→逐元素收集为 `SliceValidationError` | `binding/default_validator.go:47` |
| 内部校验入口 | 包内 `validate(obj)` 在 `Validator==nil` 时跳过，否则调 `ValidateStruct` | `binding/binding.go:122` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `Binding` | `binding/binding.go:32` | 最基础绑定契约：`Name()` 与 `Bind(*http.Request, any) error` |
| `BindingBody` | `binding/binding.go:39` | 在 `Binding` 上追加 `BindBody([]byte, any)`，供 `ShouldBindBodyWith` 复用已读 body |
| `BindingUri` | `binding/binding.go:46` | 不从 `*http.Request` 读，改从 `map[string][]string`（路由 Param）绑定 |
| `StructValidator` | `binding/binding.go:55` | 校验引擎抽象；`Engine()` 返回底层引擎以便注册自定义校验规则 |
| `Validator`（包级 var） | `binding/binding.go:72` | 默认实现 `&defaultValidator{}`，用户可整体替换为 nil 关闭校验 |
| `defaultValidator` | `binding/default_validator.go:16` | 持有 `*validator.Validate`，`once sync.Once` 保证并发安全单例初始化 |
| `SliceValidationError` | `binding/default_validator.go:21` | `[]error` 聚合，`Error()` 以 `[i]: msg` 换行拼接 |
| `Default` | `binding/binding.go:95` | 唯一的分派入口，Context 的 `ShouldBind` 调用它 |

## 3. 关键调用链

1. **自动分派绑定**：`Context.ShouldBind(obj)` → `binding.Default(c.Request.Method, c.ContentType())` 选出绑定器 → `c.ShouldBindWith(obj, b)` → `b.Bind(c.Request, obj)`。
   - `context.go:862` 调 `binding.Default`；`context.go:943` 调 `b.Bind(c.Request, obj)`。
2. **MIME 分派分支**：`Default` 先判 `method == GET` 返回 `Form`（`binding/binding.go:96`），再 switch `contentType`：`MIMEJSON→JSON`、`MIMEXML/MIMEXML2→XML`、`MIMEPROTOBUF→ProtoBuf`、`MIMEMSGPACK/MIMEMSGPACK2→MsgPack`、`MIMEYAML/MIMEYAML2→YAML`、`MIMETOML→TOML`、`MIMEMultipartPOSTForm→FormMultipart`、`MIMEBSON→BSON`、default→`Form`（`binding/binding.go:100-119`）。
3. **结构校验**：各绑定器解码完成后调用包内 `validate(obj)`（`binding/binding.go:122`）→ `defaultValidator.ValidateStruct` 按 Kind 分流 → Struct 走 `validateStruct` → `v.validate.Struct(obj)`（`binding/default_validator.go:79-81`），底层引擎在 `lazyinit` 中 `SetTagName("binding")`（`binding/default_validator.go:96`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| `binding` struct tag | 校验规则标签名，如 `binding:"required"` | `binding/default_validator.go:96`（`SetTagName("binding")`） |
| `Validator` 包级变量 | 默认 `&defaultValidator{}`；置 nil 则 `validate` 直接返回 nil 关闭全部校验 | `binding/binding.go:72`、`binding/binding.go:123` |
| `Default()` 的 GET 分支 | GET 请求无视 Content-Type 一律走 Form | `binding/binding.go:96` |
| `Engine()` 返回值 | 返回 `*validator.Validate`，供用户注册自定义校验器/结构体级校验 | `binding/default_validator.go:88` |

## 5. 错误与重试语义

- 绑定/校验本身无重试逻辑：解码或校验出错即把 error 向上抛给 Context 层。
- `Context.MustBindWith`（`context.go:833`）负责把 error 转成 HTTP 响应：`http.MaxBytesError` 走 `413`，其余走 `400`，并 `AbortWithError` + `SetType(ErrorTypeBind)`。
- `ValidateStruct` 对 Slice/Array **不短路**：逐元素收集全部错误到 `SliceValidationError`，全部跑完后一并返回（`binding/default_validator.go:61-72`）。
- 非 struct / 非 slice/array 类型（如 map、基础类型）直接返回 nil，不报错（`binding/default_validator.go:73`）。
- `defaultValidator` 注释承诺"永不 panic"：`ValidateStruct` 对任意输入类型都有分支兜底。

## 6. 并发细节

- `defaultValidator.lazyinit` 使用 `sync.Once`（`binding/default_validator.go:93`），保证 `*validator.Validate` 并发安全地只初始化一次；`*validator.Validate` 本身文档声明并发可读。
- 本叶子无 goroutine 启停、无 channel/队列；所有绑定在请求处理 goroutine 内同步完成。
- 包级注册表 `JSON/Form/...` 是不可变静态实例，初始化后只读，无锁。
- context.Context 不在本包内传播；校验是纯函数式的反射+规则判定，不依赖取消信号。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `binding/binding.go`：接口契约、MIME 常量、注册表、`Default` 分派、`validate` 入口。
- `binding/default_validator.go`：`StructValidator` 默认实现与类型分流。

**Out-of-Scope（不在本仓库源码内）**
- `github.com/go-playground/validator/v10`：真正的规则校验引擎，第三方依赖，**不在本仓库源码内**。
- 各具体绑定器的解码实现（JSON/XML/Form/...）见 `body-bindings`、`form-bindings` 叶子。
- HTTP 状态码映射与 `ErrorTypeBind` 错误类型见 Context/errors 子系统，不在本叶子。

## 8. 与相邻子系统交互

- 上游：`Context.ShouldBind/ShouldBindJSON/MustBindWith`（`context.go`）→ 调 `Default` 选绑定器、调 `validate`。
- 本叶子 → 下游：把 `StructValidator` 接口暴露给框架层；`Engine()` 让用户拿到 validator 引擎做定制。
- 平行叶子：`body-bindings`（各 `XxxBinding.Bind` 末尾调本叶子的 `validate`）、`form-bindings`（映射完成后同样调 `validate`）。

## 9. 语言专项适配口径（Go）

- **接口定义在消费侧**：`Binding/StructValidator` 定义在 `binding` 包，实现（各 `xxxBinding{}`、`defaultValidator`）也在同包，属"接口与实现同包"的经典 Go 风格；Context 仅依赖 `binding.Binding` 接口，不依赖具体类型，满足依赖倒置。
- **无 K8s 控制器模式**：gin 是 HTTP 库，本叶子无 Reconcile/informer/workqueue。
- **懒加载单例**：`sync.Once` 初始化 validator 引擎，避免包级 init 时的反射注册开销；`Engine()` 暴露内部引擎是 Go 库常见的"可观测+可扩展"做法。
- **internal 边界**：本叶子不依赖 `internal/`；下游 form_mapping 用到 `internal/bytesconv`（见 form-bindings）。
- **单二进制库形态**：gin 不产二进制，本叶子随主库一起被用户 `import`。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 绑定注册表与校验器架构图 | `binding-registry-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/binding-registry-architecture.json`。本叶子不补 sequence/dataflow：`Default` 是单函数 switch 分派、校验是纯反射调用，时序已在第 3 节文字化，无需单独时序图。
