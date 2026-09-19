# Form/Query/Uri/Header 绑定与反射映射（form-bindings）

> 本文是 `request-binding/` 域下的叶子子系统文档。域级总览见 `../request-binding.md`。
> 本文展开 **非 body 来源**（form 表单、query 串、uri 路由参数、header）绑定器与 `mapFormByTag` 反射映射机制；
> body 绑定见 `../body-bindings/`，分派与校验接口见 `../binding-registry/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| form 绑定 | `formBinding`：`ParseForm` + `ParseMultipartForm(defaultMemory)` 后 `mapForm(obj, req.Form)` | `binding/form.go:24` |
| form-urlencoded 绑定 | `formPostBinding`：只解析并绑定 `req.PostForm`（不含 query） | `binding/form.go:41` |
| multipart 绑定 | `formMultipartBinding`：`ParseMultipartForm` 后用 `multipartRequest` setter 支持文件上传 | `binding/form.go:55` |
| query 绑定 | `queryBinding`：`req.URL.Query()` 取出 `map[string][]string` 后 `mapForm` | `binding/query.go:15` |
| uri 绑定 | `uriBinding`：唯一实现 `BindingUri`，从路由 Param 的 map 绑定，走 `uri` tag | `binding/uri.go:13` |
| header 绑定 | `headerBinding`：把 `req.Header` 按 `header` tag 绑定，key 经 `CanonicalMIMEHeaderKey` 规范化 | `binding/header.go:19`、`binding/header.go:35` |
| 反射映射核心 | `mapFormByTag`：按指定 tag 遍历 struct 字段并设值；支持嵌套 struct、匿名字段、指针 | `binding/form_mapping.go:46` |
| setter 抽象 | `setter` 接口 + `formSource`/`headerSource`/`multipartRequest` 三种实现，统一 `TrySet` | `binding/form_mapping.go:66`、`binding/multipart_form_mapping.go:27` |
| 类型转换 | `setWithProperType` 覆盖 int/uint/bool/float/string/time.Duration/time.Time/map/struct(JSON) | `binding/form_mapping.go:323` |
| tag 选项 | `default`、`parser`、`collection_format`(csv/ssv/tsv/pipes)、`time_format`/`time_utc`/`time_location` | `binding/form_mapping.go:163`、`binding/form_mapping.go:213`、`binding/form_mapping.go:434` |
| 自定义解析 | `BindUnmarshaler` 接口与 `encoding.TextUnmarshaler` 支持 | `binding/form_mapping.go:183`、`binding/form_mapping.go:201` |
| multipart 文件 | `multipartRequest.TrySet` 把 `[]*multipart.FileHeader` 绑定到 FileHeader/切片/数组 | `binding/multipart_form_mapping.go:35` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `formBinding`/`formPostBinding`/`formMultipartBinding` | `binding/form.go:14` | 三个 form 变体，差别在解析哪个 map 与是否处理文件 |
| `mapForm`/`mapURI`/`MapFormWithTag` | `binding/form_mapping.go:32`、`:36`、`:40` | 对外入口，最终都调 `mapFormByTag(ptr, form, tag)` |
| `setter` 接口 | `binding/form_mapping.go:66` | `TrySet(value, field, key, opt)` 抽象"从哪种来源取值" |
| `formSource` | `binding/form_mapping.go:70` | `map[string][]string` 的 setter，供 form/query 复用 |
| `headerSource` | `binding/header.go:31` | header setter，取值前规范化 key |
| `multipartRequest` | `binding/multipart_form_mapping.go:14` | multipart setter，优先取 `MultipartForm.File[key]` 文件，否则走 Value |
| `mapping` | `binding/form_mapping.go:84` | 递归遍历 reflect 值，处理指针分配、匿名字段、未导出字段跳过 |
| `setByForm`/`setWithProperType` | `binding/form_mapping.go:245`、`:323` | 按 reflect Kind 分派取值与类型转换 |
| `defaultMemory` | `binding/form.go:12` | `32 << 20`（32MB）multipart 内存阈值 |

## 3. 关键调用链

1. **form 绑定主路径**：`formBinding.Bind` → `req.ParseForm()`（`binding/form.go:25`）→ `req.ParseMultipartForm(defaultMemory)`（`binding/form.go:28`，`ErrNotMultipart` 视为可接受）→ `mapForm(obj, req.Form)`（`binding/form.go:31`）→ `mapFormByTag` → `mappingByPtr` → `mapping` 递归。
2. **字段级设值**：`mapping` 对每字段调 `tryToSetValue` 解析 tag（`binding/form_mapping.go:145`）→ 解析 `default=`/`parser=` 选项 → `setter.TrySet` → `setByForm`（`binding/form_mapping.go:245`）→ `setWithProperType` 按 Kind 转 `strconv` 设值。
3. **multipart 文件绑定**：`multipartRequest.TrySet`（`binding/multipart_form_mapping.go:27`）先查 `r.MultipartForm.File[key]`，非空则 `setByMultipartFormFile` 按 Ptr/Struct/Slice/Array 分派（`binding/multipart_form_mapping.go:35`）；无文件则回落到 `setByForm(..., r.MultipartForm.Value, ...)`。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| `defaultMemory` | `32 << 20`（32MB），multipart 内存超阈值落盘 | `binding/form.go:12` |
| struct tag `form`/`uri`/`header`/`query` | 字段映射键名；`-` 表示忽略该字段 | `binding/form_mapping.go:85` |
| tag 选项 `default:x` | 缺省值；slice 会按 `;` 转 `,` 拆分 | `binding/form_mapping.go:163-173` |
| tag 选项 `parser:encoding.TextUnmarshaler` | 指定用 `TextUnmarshaler` 解析 | `binding/form_mapping.go:174` |
| `collection_format` | `multi`(默认)/`csv`/`ssv`/`tsv`/`pipes`，控制多值分隔 | `binding/form_mapping.go:213-230` |
| `time_format`/`time_utc`/`time_location` | time.Time 解析格式，默认 RFC3339 | `binding/form_mapping.go:434-487` |

## 5. 错误与重试语义

- 无重试：反射设值或类型转换出错即返回 error。
- `ParseForm`/`ParseMultipartForm` 错误直接返回；仅 formBinding 对 `http.ErrNotMultipart` 用 `errors.Is` 忽略（`binding/form.go:28`）。
- `setWithProperType` 遇未知 Kind 返回 `errUnknownType`（`binding/form_mapping.go:385`）；数组长度不匹配返回 `"%q is not valid value for %s"`（`binding/form_mapping.go:302`）。
- multipart 文件类型不支持返回 `ErrMultiFileHeader`；数组长度不符返回 `ErrMultiFileHeaderLenInvalid`（`binding/multipart_form_mapping.go:20-23`）。
- 未导出字段（`sf.PkgPath != "" && !sf.Anonymous`）静默跳过而非报错（`binding/form_mapping.go:124`）。
- 缺失键且无 `default` 时 `setByForm` 返回 `(false, nil)`，表示"未设置"，不视为错误（`binding/form_mapping.go:247`）。

## 6. 并发细节

- `mapFormByTag` 等都是无状态纯函数，入参为反射值，并发安全。
- multipart 解析产物 `req.Form`/`req.MultipartForm` 由 `net/http` 管理，请求内只读。
- 本叶子无 goroutine/channel/锁；`Context` 的 `queryCache/formCache`（见 facts）是在 Context 层缓存解析结果，不在本包。
- 指针字段为 nil 时，`mapping` 内部 `reflect.New` 新建并在确实设值后才 `value.Set`（`binding/form_mapping.go:94-104`），避免分配无谓对象。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `binding/form.go`、`form_mapping.go`、`query.go`、`uri.go`、`header.go`、`multipart_form_mapping.go`。

**Out-of-Scope（不在本仓库源码内）**
- `net/http` 的 `ParseForm`/`ParseMultipartForm`、`mime/multipart` 标准库能力。
- `codec/json`（form_mapping 中 struct/map 字段用 `json.API.Unmarshal` 二次解析，`binding/form_mapping.go:376`）见 `codec-json` 叶子。
- 路由 Param 来源（`Context.Params`）见路由树子系统。

## 8. 与相邻子系统交互

- 上游：Context 的 `ShouldBindQuery/ShouldBindUri/ShouldBindHeader` 等 → 对应 binding 的 `Bind`/`BindUri`。
- 本叶子 → 下游：`form_mapping.go` 用 `codec/json.API` 解析 struct/map 字段；末尾统一调 `binding-registry` 的 `validate`。
- `multipartRequest` 包装 `*http.Request`，是 setter 接口的第三态实现（区别于 formSource/headerSource）。

## 9. 语言专项适配口径（Go）

- **反射递归遍历 struct**：`mapping`（`binding/form_mapping.go:84`）是 gin 手撸的轻量反射映射器，处理指针解引用、匿名字段提升、未导出字段跳过，避免引入外部反射库。
- **策略接口 + 多实现**：`setter` 接口把"取值来源"抽象为三种实现（form/header/multipart），映射算法只依赖接口，体现 Go 面向接口编程。
- **自定义反序列化接口**：`BindUnmarshaler` 与标准库 `encoding.TextUnmarshaler` 双轨支持，是 Go 库扩展点惯例。
- 无 K8s 控制器模式；无 internal 越界（仅用 `internal/bytesconv` 做零拷贝）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Form/Query/Uri/Header 绑定架构图 | `form-bindings-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/form-bindings-architecture.json`。本叶子不补 sequence：反射映射是递归同步过程，主路径已在第 3 节文字化。
