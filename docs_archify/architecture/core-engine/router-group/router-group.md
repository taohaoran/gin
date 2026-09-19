# 路由分组与注册（router-group）

> 本文是 `core-engine` 域下的叶子子系统文档。域级总览见 `../core-engine.md`。
> 本文只展开 **RouterGroup 的结构、接口抽象、分组嵌套与 basePath 拼接、方法注册族、中间件合并、静态文件挂载**，不重复展开路由树如何插入（见 `../../router-tree/radix-tree/radix-tree.md`）与 Engine 如何持有树（见 `../engine-lifecycle/engine-lifecycle.md`）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。主源码文件：`routergroup.go`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 分组对象 | `RouterGroup{Handlers, basePath, engine, root}`，嵌套子组继承父组中间件与前缀 | `routergroup.go:55` |
| 接口抽象 | `IRouter`（含 `Group`）/ `IRoutes`（注册与静态方法族） | `routergroup.go:27`、`routergroup.go:33` |
| 中间件挂载 | `Use(middleware...)` 追加到 `group.Handlers`，返回 `returnObj()` | `routergroup.go:65` |
| 创建子组 | `Group(relativePath, handlers...)`：合并父组中间件 + 拼 basePath | `routergroup.go:72` |
| 取当前前缀 | `BasePath()` 返回 `group.basePath` | `routergroup.go:82` |
| 统一注册入口 | `handle(method, relativePath, handlers)`：拼绝对路径 + 合并 handler + `engine.addRoute` | `routergroup.go:86` |
| 自定义方法注册 | `Handle(method, path, handlers)`，正则校验方法名为全大写 ASCII，否则 panic | `routergroup.go:105`、`routergroup.go:16` |
| 方法注册族 | GET/POST/PUT/PATCH/DELETE/OPTIONS/HEAD/QUERY 各一行转 `handle` | `routergroup.go:113`–`routergroup.go:152` |
| 全方法 / 指定方法 | `Any`（遍历 `anyMethods` 9 种）、`Match(methods, path, ...)` | `routergroup.go:156`、`routergroup.go:19`、`routergroup.go:165` |
| 单文件挂载 | `StaticFile`/`StaticFileFS`：禁止路径参数，注册 GET+HEAD 调 `c.File`/`c.FileFromFS` | `routergroup.go:175`、`routergroup.go:184`、`routergroup.go:190` |
| 目录挂载 | `Static`/`StaticFS`：`http.FileServer + http.StripPrefix`，挂 `/.../*filepath` | `routergroup.go:206`、`routergroup.go:213`、`routergroup.go:226` |
| handler 合并 | `combineHandlers`：父组中间件在前、路由 handler 在后，断言不超过 `abortIndex` | `routergroup.go:251` |
| 路径拼接 | `calculateAbsolutePath = joinPaths(basePath, relativePath)` | `routergroup.go:260` |
| 返回对象 | `returnObj()`：根组返回 `engine`，子组返回 `group` | `routergroup.go:264` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `RouterGroup` struct | `routergroup.go:55` | 分组：持中间件链、前缀、指回 engine、是否根组 |
| `IRouter` interface | `routergroup.go:27` | 组合 `IRoutes` + `Group`，Engine 与 RouterGroup 都实现它（`var _ IRouter = ...` 在 `routergroup.go:62`、`gin.go:191`） |
| `IRoutes` interface | `routergroup.go:33` | 注册/中间件/静态方法全集，链式调用返回 `IRoutes` |
| `regEnLetter` | `routergroup.go:16` | `^[A-Z]+$` 正则，校验 HTTP 方法名 |
| `anyMethods` | `routergroup.go:19` | `Any` 遍历的 9 种标准方法常量数组 |
| `combineHandlers` | `routergroup.go:251` | 合并父组中间件与路由 handler，保证执行顺序 |
| `createStaticHandler` | `routergroup.go:226` | 用 `http.StripPrefix` 包 `http.FileServer`，`OnlyFilesFS` 时目录返回 404 |

## 3. 关键调用链

**链路 A：注册一条路由（以 GET 为例）**

1. 开发者调 `group.GET("/user/:id", h)` → `GET` 转调 `handle(http.MethodGet, relativePath, handlers)`（`routergroup.go:118` → `routergroup.go:86`）。
2. `handle` 先 `calculateAbsolutePath`（`joinPaths(group.basePath, relativePath)`，`routergroup.go:260`）拼出绝对路径。
3. `combineHandlers(handlers)`：把 `group.Handlers`（父组+全局中间件）拷到前，路由 handler 拷到后，并 `assert1(finalSize < abortIndex, "too many handlers")`（`routergroup.go:251-257`）。
4. 调 `group.engine.addRoute(method, absolutePath, mergedHandlers)`（`routergroup.go:89`）进入路由树（见 radix-tree 叶子）。
5. `returnObj()`：若 `group.root` 为真返回 `engine`（链式），否则返回子组 `group`（`routergroup.go:264-269`）。

**链路 B：嵌套子组继承中间件与前缀**

1. `v1 := r.Group("/v1")`：`Group` 用 `combineHandlers(nil)` 复制父组中间件，`basePath = calculateAbsolutePath("/v1")`（`routergroup.go:73-77`）。
2. 在 `v1` 上注册的路由，其 `group.basePath` 已是 `/v1`，中间件已含父组 Use 的全部中间件——子组"继承"了前缀与中间件。

**链路 C：静态目录挂载**

1. `Static(relativePath, root)` → `StaticFS(relativePath, Dir(root, false))`（`routergroup.go:206-207`）。
2. `StaticFS` 先断言 relativePath 不含 `:`/`*`（`routergroup.go:214`），`urlPattern = path.Join(relativePath, "/*filepath")`，对 GET/HEAD 各注册一次（`routergroup.go:218-222`）。
3. `createStaticHandler` 内 `fileServer = http.StripPrefix(absolutePath, http.FileServer(fs))`，闭包里 `c.Param("filepath")` 取子路径，先 `fs.Open` 校验存在性（失败则 404 并重置 `c.index = -1` 跑 noRoute），再 `fileServer.ServeHTTP`（`routergroup.go:226-248`）。

## 4. 配置项

| 配置项 / 行为 | 默认 / 说明 | 位置 |
|------|------|------|
| `group.Handlers` | 该组累积的中间件链，`Use` 追加 | `routergroup.go:56`、`routergroup.go:66` |
| `group.basePath` | 组前缀，根组为 `/`（见 `gin.go:207`） | `routergroup.go:57` |
| 方法名校验 | 仅允许全大写 ASCII，否则 panic | `routergroup.go:106-108` |
| 静态路径禁参数 | relativePath 含 `:`/`*` 直接 panic | `routergroup.go:191`、`routergroup.go:214` |
| handler 数量上限 | 合并后必须 `< abortIndex`（`MaxInt8>>1`），否则 panic | `routergroup.go:253` |

## 5. 错误与重试语义

- **开发期 panic（非运行期错误）**：非法 HTTP 方法名（`Handle`）、静态路径含路径参数、handler 数量超限——都是注册期编程错误，直接 panic 让开发者尽早发现，而非运行期返回错误。
- **静态文件 404**：`createStaticHandler` 里 `fs.Open` 失败时写 404、把 `c.handlers = group.engine.noRoute` 并 `c.index = -1` 重置链位置（`routergroup.go:238-243`），实现"文件不存在则回落到路由器的 NotFound"。
- **无重试**：注册是一次性构建；运行期查表不重试（见 engine-lifecycle）。

## 6. 并发细节

- **注册期写、运行期读**：`RouterGroup` 的方法注册直接改 `engine.trees`，与 `addRoute` 一样**非并发安全**（约定所有注册在 `Run` 前单 goroutine 完成）。
- **Use 追加**：`group.Handlers = append(...)` 在注册期单线程追加，无需锁。
- **无自有 goroutine**：本叶子不 spawn goroutine；`http.FileServer` 由运行期请求 goroutine 调用。
- **context.Context**：静态处理器闭包透传 `c.Request`（自带 ctx），不额外派生。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `routergroup.go` 全文：分组/注册/静态挂载。
- `path-utils`（`joinPaths`）与 `engine.addRoute` 的调用契约。

**Out-of-Scope（不在本仓库源码内）**
- `http.FileServer` / `http.StripPrefix`（Go 标准库 `net/http`）。
- 路由树插入与重排的内部实现（radix-tree 叶子）。

**不做什么**：不实现路由树、不定义 handler 如何执行（Context.Next 负责）、不做动态路由匹配。

## 8. 与相邻子系统交互

- **业务代码 → RouterGroup**：`r.Group/GET/POST...` 注册，链式返回 `IRoutes`。
- **RouterGroup → Engine**：`group.engine.addRoute` 把合并后的 handler 链交给 Engine 的 methodTrees。
- **RouterGroup → path-utils**：`calculateAbsolutePath` 调 `joinPaths`（`utils.go:136`）。
- **RouterGroup → 标准库**：`StaticFS` 用 `http.FileServer`/`http.StripPrefix`；`fs.go` 的 `Dir` 包装目录列表开关。
- **运行期 → Context**：静态处理器闭包回调 `c.File`/`c.FileFromFS`/`c.Param("filepath")`。

## 9. 语言专项适配口径

- **接口隔离 / 依赖方向**：`IRouter`/`IRoutes` 由 `Engine` 与 `RouterGroup` 共同实现，注册方法面向接口编程，业务代码只依赖 `IRoutes`，不直接依赖 `Engine`——典型 Go 接口由消费方定义习惯的反向体现。
- **链式 Builder 模式**：所有注册方法返回 `IRoutes`（`returnObj()`），根组返回 engine、子组返回自身，实现流式注册 API。
- **注册期 vs 运行期分离**：`combineHandlers` 在注册期把中间件"烤进"每条路由的 `HandlersChain`，运行期 `Next()` 直接顺序调用，零查找——这是 gin 中间件高性能的关键设计（与 Context 的 index 游标配合）。
- **单二进制库形态**：本叶子只是注册 API，无独立入口。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 路由分组注册架构图 | `router-group-architecture.html` | architecture | standard（降档原因：diagonal/垂直边标签在 showcase 严格校验下反复触发与 group 组件重叠，按 usage-guide 第 4.6 节降 standard，render 退出码 0） |

- JSON IR 源文件位于 `json/router-group-architecture.json`。
- 本叶子**不补时序图**：注册是一次性构建动作，无"请求生命周期"式时序；`handle → combineHandlers → addRoute` 主路径已在第 3 节文字化展开。
