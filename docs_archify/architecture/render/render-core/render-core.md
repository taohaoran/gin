# 渲染核心（render-core）

> 本文是 `render` 域下的叶子子系统文档。域级总览见 `../render.md`，本文只展开**渲染抽象接口 `Render`、写响应协议与 `writeContentType` 辅助函数**；
> 各具体渲染器（JSON/XML/HTML/Redirect 等）见 `../render-formats/render-formats.md`。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| Render 接口 | 统一渲染协议：`Render(http.ResponseWriter) error` + `WriteContentType(http.ResponseWriter)` | `render/render.go:10` |
| 写响应总编排 | `Context.Render(code, r)`：写状态码、判断是否有 body、写头、调渲染、错误入 `c.Errors` | `context.go:1202` |
| Content-Type 幂等写入 | `writeContentType` 仅在未设置 `Content-Type` 时写入，避免覆盖 | `render/render.go:37` |
| HTMLRender 子接口 | `Instance(name, data) Render` 工厂，区分调试/生产模板实现 | `render/html.go:24`、`html.go:57`、`html.go:66` |
| 实现断言清单 | `render.go` 顶部 `var _ Render = (...)` 编译期断言全部渲染器实现接口 | `render/render.go:17` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Render` 接口 | `render/render.go:10` | 所有响应渲染器的最小协议 |
| `writeContentType(w, []string)` | `render/render.go:37` | 包内辅助：幂等写 `Content-Type` 响应头 |
| `HTMLRender` 接口 | `render/html.go:24` | 模板渲染工厂接口，`HTMLProduction`/`HTMLDebug` 两实现 |
| `Context.Render` | `context.go:1202` | 渲染总入口，编排状态码/头/体/错误 |

## 3. 关键调用链

### 链一：Context.Render 写响应主流程
1. `Context.Render(code, r)`（`context.go:1202`）先 `c.Status(code)` 写状态码。
2. 若 `!bodyAllowedForStatus(code)`（如 204/304 等无 body 状态），只调 `r.WriteContentType(c.Writer)` 并 `WriteHeaderNow()` 后返回（`context.go:1205`）。
3. 否则调 `r.Render(c.Writer)`；若返回 error，`c.Error(err)` 入集中错误并 `c.Abort()`（`context.go:1211`）。

### 链二：writeContentType 幂等写头
1. 各渲染器 `WriteContentType` 统一调 `writeContentType(w, contentType)`（`render/render.go:37`）。
2. 函数检查 `w.Header()["Content-Type"]`，**仅当长度为 0（未设置）时**才写入（`render/render.go:39`）——已被业务手动设置的 Content-Type 不被覆盖。

### 链三：HTMLRender 工厂
1. `Context.HTML`（`context.go:1221`）在 `engine.HTMLRender == nil` 时渲染空 `render.HTML{}`（触发"未配置"错误）；否则 `Instance(name, obj)` 产出 `HTML` 实例再交 `c.Render`（`context.go:1227`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|-------------|------|
| `Content-Type` 响应头 | 由各渲染器自带 MIME；已存在则不覆盖 | `render/render.go:39` |
| `bodyAllowedForStatus` | 决定无 body 状态是否写响应体 | `context.go:1205` |

## 5. 错误与重试语义

- **渲染错误即中止**：`r.Render` 返回 error 时 `Context.Render` 把它推入 `c.Errors` 并 `c.Abort()`（`context.go:1213`），不再重试。
- **Redirect 特例**：`Redirect.Render`（`render/redirect.go:20`）对非法状态码直接 `panic`（非 error 返回），要求状态码在 3xx 或 201。
- **HTML 未配置**：`HTML.Render` 在 `Template == nil` 时返回 `errHTMLRendererNotConfigured`（`render/html.go:54`、`html.go:94`）。

## 6. 并发细节

- **无 goroutine**：渲染在请求 goroutine 内同步写 `http.ResponseWriter`。
- **无共享可变状态**：各 Render 实现为值类型（如 `JSON{Data: obj}`），每次请求构造；`writeContentType` 只操作当前请求的 ResponseWriter 头。
- **并发写约束**：`http.ResponseWriter` 本身不支持并发写，渲染串行执行；`Context.Writer` 包装了并发安全的写入计数。

## 7. 系统边界

**In-Scope（本仓库源码内）**：`render/render.go`（接口 + writeContentType）、`render/html.go` 的 `HTMLRender` 接口与工厂骨架。

**Out-of-Scope（不在本仓库源码内）**：
- `net/http` 的 `ResponseWriter`/`Redirect`/`Error` 行为（标准库）。
- 各具体渲染器实现（render-formats 叶子）。
- `Context.Writer`（response_writer.go）的状态码/字节计数包装。

## 8. 与相邻子系统交互

- **上游**：`Context` 所有输出方法（`JSON/String/XML/Data/HTML/Redirect/...`，`context.go:1236` 起）构造 Render 实例 → 调 `Context.Render`。
- **下游**：Render 实例 → `http.ResponseWriter` 写字节与头；HTMLRender → `html/template` 执行模板。
- **与 binding 域衔接**：业务函数常"绑定失败 → 用 render.JSON(400, err) 渲染错误"。

## 9. 语言专项适配口径（Go）

- **接口即协议**：`Render` 是典型 Go 值类型接口，大量小 struct 实现它；`render.go:17` 用 `var _ Render = (*X)(nil)` 做编译期接口实现断言。
- **无并发/控制器模式**：纯同步 IO 写，无 Reconcile/informer；唯一需注意的是 `http.ResponseWriter` 的非并发写约束。
- **internal 边界**：渲染器可引用 `internal/bytesconv`（零拷贝）与 `internal/fs`（模板文件系统包装），方向向内不泄露。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| render-core 架构图 | `render-core-architecture.html` | architecture | showcase |
| Context.Render 写响应时序 | `render-core-sequence.html` | sequence | showcase（首版含 ctx 自环消息不支持落 standard，删自环、把"错误处理"并入返回消息后通过） |

- JSON IR 源：`json/render-core-architecture.json`、`json/render-core-sequence.json`。
- 省略 dataflow/lifecycle：写响应是单次同步函数调用，无数据管道加工、无单实体状态机，按资源节省原则省略。
