# HTML / Redirect / Reader 渲染器（html-redirect-reader）

> 本文是 `response-render/` 域下的叶子子系统文档。域级总览见 `../response-render.md`。
> 本文展开 HTML 模板渲染（Debug/Production 双模式）、HTTP 重定向与 io.Reader 流式透传；
> Render 接口与 JSON 变体见 `../render-interface/`，其他 markup 见 `../markup-renderers/`。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| HTMLRender 接口 | `Instance(name, data) Render`，区分调试/生产两套模板策略 | `render/html.go:24` |
| HTMLProduction | 持有预编译 `*template.Template`，`Instance` 直接产出 `HTML` | `render/html.go:30`、`render/html.go:57` |
| HTMLDebug | 持有 Files/Glob/FileSystem+Patterns，每次请求 `loadTemplate` 重新解析 | `render/html.go:36`、`render/html.go:66` |
| 模板加载三模式 | `ParseFiles` / `ParseGlob` / `ParseFS`（经 `internal/fs` 适配） | `render/html.go:78-87` |
| HTML 渲染执行 | `HTML.Render`：`Template==nil` 报错；`Name==""` 走 `Execute`，否则 `ExecuteTemplate` | `render/html.go:92` |
| Delims 分隔符 | `Delims{Left,Right}`，默认 `{{`/`}}` | `render/html.go:16` |
| Redirect 重定向 | 校验状态码区间（3xx 或 201 Created），委托 `http.Redirect`，`WriteContentType` 空实现 | `render/redirect.go:13`、`render/redirect.go:20` |
| Reader 流式透传 | `io.Copy` 把 `io.Reader` 写入 w，支持 ContentLength/Headers | `render/reader.go:14`、`render/reader.go:22` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `HTMLRender` | `render/html.go:24` | 由 Engine 持有，按运行模式选择 Debug/Production |
| `HTMLDebug` | `render/html.go:36` | 开发期模板热重载：`loadTemplate` 用 `template.Must` 每次重解析 |
| `HTMLProduction` | `render/html.go:30` | 生产期：模板只解析一次，复用 |
| `HTML` | `render/html.go:46` | 最终渲染值对象：`Template/Name/Data` |
| `Redirect` | `render/redirect.go:13` | 重定向值对象：`Code/Location/Request` |
| `Reader` | `render/reader.go:14` | 流式值对象：`ContentType/ContentLength/Reader/Headers` |

## 3. 关键调用链

1. **HTML 渲染主路径**：`Context.HTML(code, name, obj)`（`context.go:1221`）：`engine.HTMLRender==nil` 时直接 `Render(code, render.HTML{})` 并报错；否则 `engine.HTMLRender.Instance(name, obj)` 得到渲染值 → `c.Render(code, instance)`。
2. **模板解析**：`HTMLDebug.Instance` → `loadTemplate`（`render/html.go:74`）：Files 非空走 `ParseFiles`，否则 Glob 走 `ParseGlob`，否则 FileSystem+Patterns 走 `ParseFS`，都为空 `panic`（`render/html.go:88`）；解析失败由 `template.Must` 直接 panic。
3. **执行写出**：`HTML.Render`（`render/html.go:92`）：`WriteContentType` 写 `text/html; charset=utf-8` → `Template==nil` 返回 `errHTMLRendererNotConfigured` → `Execute`/`ExecuteTemplate`；Redirect 先校验 code 区间越界 `panic`（`render/redirect.go:21`）再 `http.Redirect`；Reader 先写 Content-Type/ContentLength/Headers 再 `io.Copy`（`render/reader.go:30-31`）。

## 4. 配置项

| flag / option | 默认 / 行为 | 位置 |
|------|------|------|
| `Engine.HTMLRender` | 经 `LoadHTMLGlob/LoadHTMLFiles/SetHTMLTemplate` 装配 | `gin.go:279`、`gin.go:290`、`gin.go:317` |
| `Delims` | 默认 `{{`/`}}`，可自定义 | `render/html.go:16` |
| `Content-Length` | Reader 仅在 `ContentLength>=0` 时写 | `render/reader.go:24` |
| 运行模式 | DebugMode 走 HTMLDebug（热重载），ReleaseMode 走 HTMLProduction（预编译） | `gin.go:279`、`gin.go:317` |

## 5. 错误与重试语义

- 无重试。
- **模板解析失败 panic**：`HTMLDebug.loadTemplate` 用 `template.Must`（`render/html.go:79`），语法错误直接 panic——这是有意为之，开发期尽早暴露；生产期改用 Production 预编译避免运行期 panic。
- `HTML.Render` 在 `Template==nil` 时返回 `errHTMLRendererNotConfigured`（`render/html.go:94`）而非 panic。
- `Redirect` 状态码不在 `300-308` 且非 `201` 时 `panic`（`render/redirect.go:22`）。
- `Reader.writeHeaders` 对已存在 header 跳过，不覆盖（`render/reader.go:44`）。
- 错误经 `Context.Render` 统一 `c.Error+c.Abort`。

## 6. 并发细节

- `HTMLProduction` 持有预编译 `*template.Template`，`template.Template.Execute` 文档声明并发安全。
- `HTMLDebug` 每次请求新建模板（`loadTemplate` 在 `Instance` 内调用，`render/html.go:68`），无跨请求共享可变状态；代价是每请求重新 Parse。
- `Reader` 用 `io.Copy` 流式拷贝，不一次性读入内存。
- 本叶子无 goroutine/channel/锁；模板执行在请求 goroutine 内同步完成。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `render/html.go`、`render/redirect.go`、`render/reader.go`。
- `internal/fs` 把 `http.FileSystem` 适配成 `fs.FS` 供 `ParseFS`（仓库内）。

**Out-of-Scope（不在本仓库源码内）**
- `html/template`（标准库）真正的模板引擎。
- `http.Redirect`（标准库）负责写 `Location` 头与 3xx 行为。
- `Engine.LoadHTML*` 装配见 gin.go（engine-lifecycle 子系统）。

## 8. 与相邻子系统交互

- 上游：`Context.HTML/Redirect/DataFromReader`（`context.go:1221`、`context.go:1314`）。
- 本叶子 → 下游：`html/template`、`http.Redirect`、`io.Copy`。
- `Engine` 通过 `HTMLRender` 字段注入本叶子的策略实现。

## 9. 语言专项适配口径（Go）

- **策略接口双实现**：`HTMLRender` 同一接口两实现（Debug 热重载 / Production 预编译），按运行模式选择，是 Go 库"开发/生产双轨"典型模式。
- **模板并发模型**：Go 的 `template.Template` 设计为可并发 Execute，故 Production 直接全局复用；Debug 每次重建是用性能换迭代便利。
- **panic 作为配置错误**：模板语法错误、非法 redirect 状态码用 panic 而非 error，属"启动期/编程期错误早失败"风格。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| HTML/Redirect/Reader 渲染器架构图 | `html-redirect-reader-architecture.html` | architecture | showcase（render 退出码 0） |

JSON IR 源文件：`json/html-redirect-reader-architecture.json`。本叶子不补 sequence：模板解析与执行主路径已在第 3 节文字化。
