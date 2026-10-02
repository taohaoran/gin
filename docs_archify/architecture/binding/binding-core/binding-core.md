# 绑定核心（binding-core）

> 本文是 `binding` 域下的叶子子系统文档。域级总览见 `../binding.md`，本文只展开**绑定抽象协议、表单反射映射引擎与统一校验**，
> 不重复展开各具体格式实现（见 `../binding-formats/binding-formats.md`）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 绑定抽象接口族 | `Binding` / `BindingBody` / `BindingUri` 三套接口，分别对应"从请求体绑定""从字节绑定""从路由参数 map 绑定" | `binding/binding.go:32`、`binding.go:39`、`binding.go:46` |
| 校验器抽象 | `StructValidator` 接口 + 包级单例 `Validator`，解耦校验引擎 | `binding/binding.go:55`、`binding.go:72` |
| 按 ContentType 分发 | `Default(method, contentType)` 按 GET 方法与 MIME 选择预定义 Binding 实例 | `binding/binding.go:95` |
| 预定义 Binding 实例 | `JSON/XML/Form/Query/FormPost/FormMultipart/ProtoBuf/MsgPack/YAML/Uri/Header/Plain/TOML/BSON` 包级变量 | `binding/binding.go:76` |
| 统一校验入口 | 包内 `validate(obj)` 调用 `Validator.ValidateStruct`，nil 安全 | `binding/binding.go:122` |
| 表单反射映射 | `mapForm/mapURI/MapFormWithTag` 把 `map[string][]string` 按 struct tag 反射写入目标对象 | `binding/form_mapping.go:32`、`form_mapping.go:36`、`form_mapping.go:40` |
| 递归字段遍历 | `mapping()` 递归处理指针、匿名字段、未导出字段，支持 tag `"-"` 忽略 | `binding/form_mapping.go:84` |
| tag 选项解析 | 解析 `form:"name,default=x,parser=..."`，支持 `default` 与 `parser` 选项 | `binding/form_mapping.go:145` |
| 类型化赋值 | `setWithProperType` 覆盖 int/uint/bool/float/string/time.Time/Duration/嵌套 struct/map | `binding/form_mapping.go:323` |
| 自定义类型钩子 | `BindUnmarshaler` 接口与 `encoding.TextUnmarshaler` 两种自定义解析扩展点 | `binding/form_mapping.go:183`、`form_mapping.go:201` |
| 集合格式 | `collection_format` tag 支持 csv/ssv/tsv/pipes 拆分多值 | `binding/form_mapping.go:213` |
| map 目标直写 | 目标为 `map[string][]string`/`map[string]string` 时直接拷贝（取最后一个值） | `binding/form_mapping.go:528` |
| 文件上传映射 | `multipartRequest` 实现 `setter`，把 `multipart.FileHeader` 绑定到字段 | `binding/multipart_form_mapping.go:14`、`multipart_form_mapping.go:27` |
| Header 绑定 | `headerSource` 把 tag key 经 `CanonicalMIMEHeaderKey` 规范化后复用 `setByForm` | `binding/header.go:31`、`header.go:35` |
| URI 参数绑定 | `uriBinding.BindUri` 调用 `mapURI` 后校验 | `binding/uri.go:13` |
| 默认校验器 | `defaultValidator` 懒加载 `go-playground/validator/v10`，tag 名为 `binding` | `binding/default_validator.go:47`、`default_validator.go:93` |
| 切片聚合校验 | `SliceValidationError` 聚合切片逐元素校验错误，按行拼接 | `binding/default_validator.go:21` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Binding` 接口 | `binding/binding.go:32` | 最基础绑定协议：`Name()` + `Bind(*http.Request, any) error` |
| `BindingBody` 接口 | `binding/binding.go:39` | 内嵌 `Binding`，增加 `BindBody([]byte, any)`，供可复用 body 绑定 |
| `BindingUri` 接口 | `binding/binding.go:46` | 仅 `Name()` + `BindUri(map[string][]string, any)`，不走 *http.Request |
| `StructValidator` 接口 | `binding/binding.go:55` | 校验引擎抽象：`ValidateStruct(any) error` + `Engine() any` |
| `Validator` 包级单例 | `binding/binding.go:72` | 默认指向 `&defaultValidator{}`，可被整体替换 |
| `setter` 接口 | `binding/form_mapping.go:66` | 抽象"按 key 取值写入字段"策略；`formSource`/`headerSource`/`multipartRequest` 三实现 |
| `formSource` | `binding/form_mapping.go:70` | `map[string][]string` 的 setter 包装（query/form/postform） |
| `multipartRequest` | `binding/multipart_form_mapping.go:14` | `*http.Request` 的 setter 包装，优先取文件、回退取普通表单值 |
| `setOptions` | `binding/form_mapping.go:138` | 携带 `default` 默认值与 `parser` 解析器选项 |
| `BindUnmarshaler` | `binding/form_mapping.go:183` | 用户自定义类型钩子：`UnmarshalParam(string) error` |
| `defaultValidator` | `binding/default_validator.go:16` | 用 `sync.Once` 懒初始化底层 validator，实现 `StructValidator` |
| `SliceValidationError` | `binding/default_validator.go:21` | `[]error`，实现 `error` 接口聚合切片校验错误 |

## 3. 关键调用链

### 链一：ShouldBindQuery 的表单反射绑定（Context → queryBinding → 反射映射）
1. `Context.ShouldBindQuery(obj)` 调用 `c.ShouldBindWith(obj, binding.Query)`，后者执行 `b.Bind(c.Request, obj)`（`context.go:942`，即 `binding-formats` 叶子的 `queryBinding.Bind`，`binding/query.go:15`）。
2. `queryBinding.Bind` 取 `req.URL.Query()` 得 `map[string][]string`，调用 `mapForm(obj, values)` → `mapFormByTag(ptr, form, "form")`（`binding/form_mapping.go:36`）。
3. `mapFormByTag` 先判断目标是否为 `map[string]string`（走 `setFormMap`，`form_mapping.go:528`），否则走 `mappingByPtr(ptr, formSource(form), tag)`（`form_mapping.go:62`）。
4. `mapping`（`binding/form_mapping.go:84`）递归遍历：指针则按需 `reflect.New` 分配；匿名字段/struct 则逐字段下钻；命中叶子字段时调 `tryToSetValue`（`form_mapping.go:145`）解析 tag。
5. `tryToSetValue` 解析出 tag 名与 `default`/`parser` 选项后，调 `setter.TrySet` → `formSource.TrySet` → `setByForm`（`binding/form_mapping.go:245`），按 kind 分发到 `setSlice/setArray/setWithProperType` 完成赋值。
6. 回到 `queryBinding.Bind` 末尾调用 `validate(obj)`（`binding/binding.go:122`）→ `defaultValidator.ValidateStruct`（`binding/default_validator.go:47`）。

### 链二：defaultValidator 的懒初始化与类型分发校验
1. `validate(obj)` 判空 `Validator` 后调用 `Validator.ValidateStruct(obj)`（`binding/binding.go:122`）。
2. `ValidateStruct`（`binding/default_validator.go:47`）用 `reflect` 按 kind 分发：Ptr/Struct 走 `validateStruct`；Slice/Array 逐元素递归并聚合为 `SliceValidationError`；其余类型直接返回 nil。
3. `validateStruct`（`binding/default_validator.go:79`）先 `lazyinit()`（`default_validator.go:93`，`sync.Once` 创建 `validator.New()` 并 `SetTagName("binding")`），再 `v.validate.Struct(obj)`。

### 链三：multipart 表单的文件 + 普通值混合绑定
1. `formMultipartBinding.Bind`（`binding/form.go:55`）先 `ParseMultipartForm(32<<20)`，再 `mappingByPtr(obj, (*multipartRequest)(req), "form")`。
2. `multipartRequest.TrySet`（`binding/multipart_form_mapping.go:27`）先查 `r.MultipartForm.File[key]`：有文件走 `setByMultipartFormFile`（`multipart_form_mapping.go:35`，支持 `*FileHeader`/`FileHeader`/切片/数组）；无文件回退到 `setByForm(value, field, r.MultipartForm.Value, key, opt)`（`multipart_form_mapping.go:32`），与普通表单共用类型解析。

## 4. 配置项

| 配置 / tag | 默认 / 行为 | 位置 |
|------------|-------------|------|
| `form` / `uri` / `header` struct tag | tag 值即字段在来源 map 中的 key；空则用字段名；`"-"` 忽略该字段 | `binding/form_mapping.go:85`、`form_mapping.go:152` |
| `form:"...,default=v"` | 来源缺该 key 时用默认值；切片默认值 `;` 会转 `,` | `binding/form_mapping.go:163` |
| `form:"...,parser=encoding.TextUnmarshaler"` | 用 `encoding.TextUnmarshaler` 解析单值 | `binding/form_mapping.go:174` |
| `collection_format` | `multi`(默认)/`csv`/`ssv`/`tsv`/`pipes`，控制多值拆分分隔符 | `binding/form_mapping.go:213` |
| `time_format` | 默认 `time.RFC3339`；支持 `unix/unixmilli/unixmicro/unixnano` | `binding/form_mapping.go:435` |
| `time_utc` / `time_location` | 控制 time.Time 解析时区 | `binding/form_mapping.go:469`、`form_mapping.go:473` |
| `binding` struct tag | validator 校验规则 tag 名（非默认 `validate`） | `binding/default_validator.go:96` |
| `binding.Validator` | 包级变量，可整体替换校验引擎为自定义实现 | `binding/binding.go:72` |

## 5. 错误与重试语义

- **无重试**：绑定是同步一次性解析，任何解码/反射/校验错误直接返回给 `Context`，框架不做重试。
- **MustBindWith 错误映射**：`Context.MustBindWith`（`context.go:833`）用 `errors.As` 识别 `*http.MaxBytesError` → 返回 `413 Request Entity Too Large`；其余绑定错误统一 `400 Bad Request` 并 `AbortWithError(...).SetType(ErrorTypeBind)`。
- **反射错误**：类型不支持时返回 `errUnknownType`（`form_mapping.go:23`）；map 目标类型不符返回 `ErrConvertMapStringSlice`/`ErrConvertToMapString`（`form_mapping.go:26`）；数组长度不匹配返回格式化错误（`form_mapping.go:302`）；文件字段类型不支持返回 `ErrMultiFileHeader`/`ErrMultiFileHeaderLenInvalid`（`multipart_form_mapping.go:20`）。
- **校验错误**：`defaultValidator` 透传 `validator.ValidationErrors`；切片场景聚合为 `SliceValidationError`，按 `[i]: msg` 换行拼接（`default_validator.go:24`）。
- **protobuf 特例**：`protobufBinding.BindBody` 故意不调用 `validate(obj)`（`binding/protobuf.go:37`），因为 protoc 生成结构无法加 `binding` tag。

## 6. 并发细节

- **无 goroutine**：整个绑定链路在请求处理 goroutine 内同步执行，不创建额外协程，无 channel/workqueue。
- **懒初始化并发安全**：`defaultValidator.once sync.Once`（`default_validator.go:17`）保证底层 validator 引擎在首次校验时单次初始化，后续并发请求共享同一引擎——这是本叶子唯一的临界区。
- **无共享可变状态**：各 `jsonBinding{}`/`formBinding{}` 等均为无状态空结构体单例；反射映射全程操作调用方传入的目标 `obj`，包级无其它可变状态。
- **context 传递**：绑定不使用 `context.Context`，超时/取消由 `net/http` server 在更上层控制；请求体读取（`io.ReadAll`）不受 ctx 取消联动。
- **body 复用**：`Context.ShouldBindBodyWith`（`context.go:951`）把读过的 body 字节缓存进 `c.Set(BodyBytesKey, body)`，允许同一请求多次 `BindBody`，避免 body 流被消费后无法重读。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `binding/binding.go`：接口族、`Default` 分发、预定义实例、`validate` 入口
- `binding/form_mapping.go`：反射映射引擎、类型解析、tag 选项、map 直写
- `binding/multipart_form_mapping.go`：文件上传字段绑定
- `binding/default_validator.go`：默认校验器封装
- `binding/uri.go`、`binding/header.go`：URI/Header 两个 setter 变体

**Out-of-Scope（不在本仓库源码内）**
- `github.com/go-playground/validator/v10`：底层 struct 校验引擎，第三方库
- `github.com/gin-gonic/gin/codec/json`：可插拔 JSON codec（标准库/go-json/jsoniter/sonic 四实现），被 `setWithProperType` 用于嵌套 struct/map 解析
- `net/http` 的 `ParseForm/ParseMultipartForm`：表单与 multipart 解析本身
- 各具体格式解码器（JSON/XML/YAML/TOML/msgpack/protobuf/bson/plain）属 `binding-formats` 叶子
- 路由参数 `c.Params` 的来源（基数树匹配）属 routing 域

## 8. 与相邻子系统交互

- **上游**：根包 `Context`（`context.go`）→ `ShouldBindWith/MustBindWith/ShouldBindUri/ShouldBindBodyWith` → 本叶子接口族；`Context.ShouldBind` 先调 `binding.Default`（`binding.go:95`）选实例。
- **下游**：本叶子 → `codec/json.API`（嵌套 struct/map 反序列化，`form_mapping.go:376`）；→ `validator/v10`（struct 校验，`default_validator.go:81`）。
- **失败 → render 域**：绑定错误经 `Context.MustBindWith` 转成 400/413 并 `c.Error`，业务处理函数通常再用 `c.JSON(400, ...)`（render 域）渲染错误响应——binding 失败与 render 成功在此衔接。

## 9. 语言专项适配口径（Go）

- **并发模型**：非 K8s 控制器模式，无 Reconcile/informer/workqueue。唯一同步原语是 `defaultValidator.once sync.Once`（`default_validator.go:94`）的懒加载单例；其余为请求内同步反射，无 goroutine、无锁竞争。
- **接口即扩展点（依赖倒置）**：`Binding/BindingBody/BindingUri/StructValidator/setter/BindUnmarshaler` 均由消费方（本包 Context 桥接层）定义接口、由各格式实现——标准 Go 面向接口编程；用户可替换 `binding.Validator` 或实现 `BindUnmarshaler` 扩展自定义类型。
- **无多二进制**：`binding/` 是纯库子包，无 `cmd/` 入口；通过 build tag `nomsgpack` 条件编译裁剪 msgpack 依赖（见 `binding_nomsgpack.go`）。
- **internal 边界**：本包 import 仓库内 `internal/bytesconv`（零拷贝 string↔[]byte）与 `codec/json`；`internal/` 仅仓库内可导入，方向清晰、无环依赖。
- **可观测性**：校验错误经 `Context.Error(...).SetType(ErrorTypeBind)` 进入 gin 集中错误管理，不自行打日志。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| binding-core 架构图 | `binding-core-architecture.html` | architecture | showcase |
| 表单/查询参数绑定调用链时序 | `binding-core-sequence.html` | sequence | showcase（首版参与者标签超长落 standard，缩短为 2~4 字短标签并加 `column_fit: spread` 后通过） |

- JSON IR 源：`json/binding-core-architecture.json`、`json/binding-core-sequence.json`。
- 省略 lifecycle：本叶子无"单实体多状态迁移"语义（绑定是一次性函数调用，非状态机），按资源节省原则省略。
