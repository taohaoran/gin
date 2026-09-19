# Render 接口与 JSON 渲染变体（render-interface）

> 本文是 `response-render/` 域下的叶子子系统文档。域级总览见 `../response-render.md`。
> 本文展开 `Render` 统一接口与 JSON 系列变体；其他 markup 渲染器见 `../markup-renderers/`，
> HTML/Redirect/Reader 见 `../html-redirect-reader/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Render 接口 | `Render(w) error` + `WriteContentType(w)` 两方法，所有渲染器实现 | `render/render.go:10` |
| Content-Type 幂等写入 | `writeContentType`：仅当 header 未设 `Content-Type` 时写入，不覆盖已有值 | `render/render.go:37` |
| 编译期实现断言 | 包级 `var _ Render = ...` 列出全部 15+ 实现，保证接口满足 | `render/render.go:17` |
| JSON 渲染 | `JSON{Data}`：`WriteJSON` 写 Content-Type + Marshal + w.Write | `render/json.go:19`、`render/json.go:67` |
| IndentedJSON | `MarshalIndent` 四空格缩进，仅建议开发用 | `render/json.go:24`、`render/json.go:78` |
| SecureJSON | 数组 JSON 前注入 `Prefix`（默认 `while(1);`）防 JSON 劫持 | `render/json.go:29`、`render/json.go:94` |
| JsonpJSON | 包裹 `callback(...)`，callback 经 `template.JSEscapeString` 转义 | `render/json.go:35`、`render/json.go:117` |
| AsciiJSON | 非 ASCII rune 转义为 `\uXXXX`，输出纯 ASCII | `render/json.go:41`、`render/json.go:155` |
| PureJSON | `SetEscapeHTML(false)`，不转义 `<`/`>`/`&` | `render/json.go:46`、`render/json.go:184` |
| Context 渲染入口 | `Context.Render(code, r)`：状态码 + 空 body 状态码两分支 + 错误 `c.Error+c.Abort` | `context.go:1202` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `Render` 接口 | `render/render.go:10` | 渲染抽象：写内容体与写 Content-Type 分离两阶段 |
| `writeContentType` | `render/render.go:37` | 幂等设置，避免覆盖用户已写的 Content-Type |
| `JSON`/`IndentedJSON`/`SecureJSON`/`JsonpJSON`/`AsciiJSON`/`PureJSON` | `render/json.go:19-48` | 仅持有 `Data`（及 Prefix/Callback）的值对象 |
| `WriteJSON` | `render/json.go:67` | 公共 JSON 写出：Content-Type → `json.API.Marshal` → `w.Write` |
| 三个 Content-Type 常量 | `render/json.go:51` | `jsonContentType`(含 charset)、`jsonpContentType`(javascript)、`jsonASCIIContentType`(无 charset) |

## 3. 关键调用链

1. **c.JSON 主路径**：`Context.JSON(code, obj)`（`context.go:1260`）→ `c.Render(code, render.JSON{Data:obj})` → `Context.Render`（`context.go:1202`）：先 `c.Status(code)`，若 `bodyAllowedForStatus(code)` 为 false 则只 `WriteContentType` + `WriteHeaderNow` 后返回，否则 `r.Render(c.Writer)`。
2. **JSON.Render**：`JSON.Render` → `WriteJSON(w, r.Data)`（`render/json.go:57`、`render/json.go:67`）→ `writeContentType(w, jsonContentType)` → `json.API.Marshal(obj)` → `w.Write(jsonBytes)`。
3. **两阶段错误处理**：`Context.Render` 中 `r.Render` 返回非 nil error 时 `_ = c.Error(err)` 并 `c.Abort()`（`context.go:1211-1214`）；SecureJSON 在确认是数组（`[` 前缀 `]` 后缀，`render/json.go:101`）后才写 Prefix。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| `Engine.secureJSONPrefix` | 默认 `"while(1);"`，可经 `SecureJsonPrefix` 改 | `gin.go:224`、`gin.go:266` |
| `Content-Type` 已存在 | `writeContentType` 不覆盖用户自设值 | `render/render.go:39` |
| JSON 编码后端 | 统一走 `codec/json.API`，见 `codec-json` 叶子 | `render/json.go:14` |
| 空 body 状态码（204/304/1xx） | `bodyAllowedForStatus` 为 false 时不写 body | `context.go:1205` |

## 5. 错误与重试语义

- 无重试：Marshal 或 `w.Write` 出错即返回 error。
- `Context.Render` 把渲染 error 压入 `c.Errors`（类型由调用方约定）并 `Abort`，避免后续 handler 继续写已部分提交的响应。
- SecureJSON：先 Marshal 成功后才判断数组前缀，写入 Prefix 失败会返回 error（`render/json.go:103`）。
- JsonpJSON：`Callback==""` 时退化为纯 JSON 写出（`render/json.go:124`）。
- 错误不回滚已写入的 header：HTTP 一旦 `WriteHeader` 提交后无法改状态码。

## 6. 并发细节

- 所有渲染器值对象无共享可变状态，可并发使用。
- `writeContentType` 读-改 `w.Header()` map，依赖 `net/http` 自身对 header map 的约定；同一响应只由单 goroutine 写。
- 本叶子无 goroutine/channel/锁；渲染在请求处理 goroutine 内同步完成。
- `Context.Render` 无 context.Context 超时介入；写出阻塞由 net/http 连接层控制。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `render/render.go`、`render/json.go`。

**Out-of-Scope（不在本仓库源码内）**
- `codec/json` 编码后端抽象见 `codec-json` 叶子。
- `html/template.JSEscapeString`（标准库）供 JSONP callback 转义。
- `http.ResponseWriter`、`bodyAllowedForStatus` 与 `c.Error/c.Abort` 见 Context 子系统。

## 8. 与相邻子系统交互

- 上游：`Context.JSON/IndentedJSON/SecureJSON/JSONP/AsciiJSON/PureJSON`（`context.go:1260` 起）构造各渲染值对象并调 `c.Render`。
- 本叶子 → 下游：所有 JSON 变体经 `codec/json.API` 编码到 `ResponseWriter`。
- 平行：`markup-renderers`（XML/YAML/...）与 `html-redirect-reader` 同样实现 `Render` 接口。

## 9. 语言专项适配口径（Go）

- **接口 + 值对象**：渲染器是纯数据 struct，方法集实现接口；包级 `var _ Render = ...` 是 Go 编译期接口满足断言惯例。
- **两阶段提交**：`WriteContentType` 与 `Render` 分离，是因为 Go 需在写 body 前先定 header，且需支持"空 body 状态码"短路。
- **统一编码后端**：JSON 变体不直接 `encoding/json`，而走 `codec/json.API`，使 build tag 切换 sonic/jsoniter/go-json 对渲染层透明（见 codec-json）。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| Render 接口与 JSON 变体架构图 | `render-interface-architecture.html` | architecture | showcase（render 退出码 0） |
| c.JSON 渲染主路径时序图 | `render-interface-sequence.html` | sequence | showcase（render 退出码 0） |

JSON IR 源文件：`json/render-interface-architecture.json`、`json/render-interface-sequence.json`。
