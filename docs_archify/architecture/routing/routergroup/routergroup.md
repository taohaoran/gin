# 路由组与路由注册（routergroup）

> 本文是 `routing` 域下的叶子子系统文档。域级总览见 `../routing.md`（输出根 `docs_archify/architecture/`）。
> 本文只展开「路由组如何组织路径前缀、合并中间件并把路由交给基数树」，不重复展开基数树匹配
> （见 `../tree/tree.md`）与路径规范化（见 `../path/path.md`）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 路由组抽象 | `RouterGroup` 绑定一个 `basePath` 前缀与一段共享 `HandlersChain`（中间件） | `routergroup.go:55` |
| 组创建与嵌套 | `Group(relativePath, handlers...)` 派生子组：复制父组处理链、拼接绝对前缀、共享同一 `engine` | `routergroup.go:72` |
| 中间件注册 | `Use(middleware...)` 向当前组追加中间件，返回 `IRoutes` 以便链式调用 | `routergroup.go:65` |
| 底层注册 | `handle(method, relativePath, handlers)` 拼绝对路径 + 合并处理链后调用 `engine.addRoute` | `routergroup.go:86` |
| 方法校验注册 | `Handle(method, path, handlers)` 用 `regEnLetter` 校验方法名必须为大写 ASCII 字母，否则 panic | `routergroup.go:105` |
| HTTP 方法快捷方法 | `GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS` 均一行转调 `handle` | `routergroup.go:113-145` |
| QUERY 方法 | RFC 10008 定义的 QUERY 方法快捷（带请求体的查询） | `routergroup.go:150` |
| 多方法批量 | `Any` 注册 9 个标准方法；`Match(methods, path)` 按指定方法列表批量注册 | `routergroup.go:156`、`routergroup.go:165` |
| 静态文件服务 | `StaticFile/StaticFileFS/Static/StaticFS` 注册单文件或目录（内部用 `http.FileServer`） | `routergroup.go:175-224` |
| 处理链合并 | `combineHandlers` 把父组中间件与本次处理函数复制拼接为一条 `HandlersChain` | `routergroup.go:251` |
| 绝对路径拼接 | `calculateAbsolutePath` 经 `joinPaths` 拼父前缀与相对路径 | `routergroup.go:260`、`utils.go:136` |
| 路由反射 | `Engine.Routes()` 遍历各方法基数树，输出 `RoutesInfo`（method/path/handler 名） | `gin.go:390` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `RouterGroup` | `routergroup.go:55` | 路由组实体，字段 `Handlers HandlersChain`（共享中间件）、`basePath string`、`engine *Engine`、`root bool` |
| `IRouter` | `routergroup.go:27` | 组合 `IRoutes` + `Group(...) *RouterGroup`，定义「组与路由」统一接口 |
| `IRoutes` | `routergroup.go:33` | 声明 `Use/Handle/GET/POST/.../Static*` 全部路由注册方法，`RouterGroup` 与 `Engine` 均实现它 |
| `regEnLetter` | `routergroup.go:16` | 预编译正则 `^[A-Z]+$`，校验自定义 HTTP 方法名合法性 |
| `anyMethods` | `routergroup.go:19` | `Any` 批量注册的 9 个标准方法常量切片 |
| `combineHandlers` | `routergroup.go:251` | 合并父组 `Handlers` 与本次 `handlers`，并断言总数 `< abortIndex` |
| `returnObj` | `routergroup.go:264` | 根组（`root=true`）返回 `engine` 自身，否则返回组自身——这是链式 API 与 `Engine` 共用接口的关键 |

## 3. 关键调用链

**链一：注册一条 GET 路由（用户 → 基数树）**
1. 用户调用 `routergroup.go:118` `(*RouterGroup).GET`，仅转调 `handle(http.MethodGet, ...)`。
2. `routergroup.go:86` `handle` 先 `calculateAbsolutePath`（`routergroup.go:260` → `joinPaths`，`utils.go:136`）拼出绝对路径，再 `combineHandlers`（`routergroup.go:251`）把组中间件与业务处理函数拼成 `HandlersChain`。
3. `handle` 调 `group.engine.addRoute`（`gin.go:364`）；`addRoute` 先 `assert1` 校验路径以 `/` 开头、方法非空、处理函数非空，再 `engine.trees.get(method)`（`gin.go:371`，线性查找 `methodTrees`）取该方法的基数树根，最终 `root.addRoute(path, handlers)`（`gin.go:377`）落入 `tree.go:135`。
4. 返回前 `addRoute` 用 `countParams`/`countSections`（`tree.go:80`、`tree.go:86`）更新 `engine.maxParams`/`maxSections`，供 Context 池预分配 Params 容量。

**链二：嵌套分组 + 中间件继承**
1. `router.Group("/api", authMiddleware)` 走 `routergroup.go:72` `Group`：`combineHandlers(handlers)` 复制父组 `Handlers` 再追加传入中间件，`basePath` 经 `calculateAbsolutePath` 拼成 `/api`。
2. 子组上再 `POST("/x", h)` 时，`handle` 再次 `combineHandlers`，最终链 = 父组中间件 + 子组中间件 + `authMiddleware` + 业务 `h`。

**链三：静态目录注册**
1. `routergroup.go:213` `StaticFS` 先断言 `relativePath` 不含 `:`/`*`（否则 panic），再 `createStaticHandler`（`routergroup.go:226`）用 `http.StripPrefix` + `http.FileServer(fs)` 包装。
2. 内部以 `path.Join(relativePath, "/*filepath")` 注册通配路由，并同时注册 `GET` 与 `HEAD` 两个方法（`routergroup.go:221-222`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `Engine.RedirectTrailingSlash` | 默认 `true`；匹配失败时按 `tsr` 结果做尾斜杠重定向（由 `tree` 叶子产出 tsr 标志） | `gin.go:211` |
| `Engine.RedirectFixedPath` | 默认 `false`；开启后大小写/斜杠修正重定向（`redirectFixedPath` 走 `findCaseInsensitivePath`） | `gin.go:212` |
| `Engine.HandleMethodNotAllowed` | 默认 `false`；开启后对同路径其他方法探测，命中则返回 405 并带 `Allow` 头 | `gin.go:213` |
| `Engine.RemoveExtraSlash` | 默认 `false`；开启后请求路径先经 `cleanPath` 规范化（见 `path` 叶子） | `gin.go:219`、`gin.go:703` |
| `Engine.UseRawPath` / `UseEscapedPath` / `UnescapePathValues` | 默认 `false`/`false`/`true`；决定匹配时用 RawPath/EscapedPath 及是否反解参数 | `gin.go:217-220` |
| 路由方法名约束 | `Handle` 要求方法名为纯大写 ASCII 字母，否则 panic | `routergroup.go:106` |

## 5. 错误与重试语义

- **注册期错误一律 panic（fail-fast），不返回 error**：方法名非法（`routergroup.go:107`）、静态路径含 `:`/`*`（`routergroup.go:192`、`routergroup.go:215`）、处理链过长（`combineHandlers` 断言 `finalSize < abortIndex`，`routergroup.go:253`）、基数树内路径冲突或重复注册（`tree.go:230`、`tree.go:243`）均直接 panic。这是刻意设计：路由注册发生在服务启动阶段，配置错误应尽早暴露，而非运行期容忍。
- **运行期不重试**：路由匹配是纯查找，无网络/IO，无重试与退避。匹配失败的兜底由 `Engine.handleHTTPRequest`（`gin.go:690`）处理：tsr 重定向、fixed-path 重定向、405、404，均一次性判定。
- **静态文件**：`createStaticHandler` 打开文件失败时写 404 并把 `c.handlers` 切到 `engine.noRoute`、重置 `c.index=-1`（`routergroup.go:239-243`），不返回错误。

## 6. 并发细节

- **路由注册不是并发安全的**：`tree.go:134` 明确注释 `Not concurrency-safe!`。全部 `Use/Group/GET/...` 注册方法必须在服务启动、监听之前单线程完成；`Engine.Run` 之后不得再注册路由。
- **运行期只读**：请求处理阶段 `handleHTTPRequest` 只并发读 `engine.trees`，不再修改树结构，因此可安全并发。
- **Context 池**：`c.params`、`c.skippedNodes` 由 `sync.Pool` 复用（`gin.go:183`），`getValue` 写入的 `Params` 容量由 `maxParams`/`maxSections` 预分配决定，避免热路径重复扩容。
- 本叶子无 goroutine 启停、无 channel；中间件链 `HandlersChain` 的顺序执行由 `Context.Next()` 驱动（见 context 域），本叶子只负责「把链拼好」。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `routergroup.go`：路由组、中间件合并、方法快捷方法、静态文件注册。
- `gin.go` 中 `Engine.addRoute`（`gin.go:364`）、`Routes()`（`gin.go:390`）、`Use`（`gin.go:340`）与路由相关配置字段。
- `utils.go` 的 `joinPaths`（`utils.go:136`）。

**Out-of-Scope（不在本仓库源码内 / 相邻叶子覆盖）**
- 基数树的插入与匹配算法（`addRoute`/`getValue`）→ 见 `../tree/tree.md`。
- 路径规范化 `cleanPath`/`removeRepeatedChar` → 见 `../path/path.md`。
- 中间件链的运行时执行（`Context.Next`/`index` 推进）→ 由 context 域覆盖。
- `net/http` 标准库的 `http.FileServer`/`http.StripPrefix`/`http.FileSystem` 为外部依赖（不在本仓库源码内）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户代码 / `Engine.New()` 后的链式 API 调用 `Use/Group/GET/...`。
- 本叶子 → 下游：`handle` → `engine.addRoute`（`gin.go:364`）→ `methodTrees.get` → `node.addRoute`（`tree.go:135`），把绝对路径与处理链写入基数树。
- 反向：`Engine.handleHTTPRequest`（`gin.go:690`）在请求期调用 `node.getValue` 取回 `handlers HandlersChain`，交回 Context 执行——本叶子注册的链正是这里被消费的对象。
- 配置：`RemoveExtraSlash` 开启时，请求路径先经 `path` 叶子的 `cleanPath` 再进基数树。

## 9. 语言专项适配口径

- **并发模型**：无标准 k8s Reconcile/informer 管道；gin 是「注册期单线程构建不可变基数树 + 运行期多 goroutine 只读查找」的客户端-服务器模式。`context.Context` 不贯穿注册期（注册无超时/取消概念），请求期由 `c.Request.Context()` 承载取消传播。
- **控制器模式差异**：无 Reconcile 幂等重入；路由冲突通过启动期 panic 而非运行期调和来保证一致性。
- **多二进制**：本仓库无 `cmd/`，是纯库；`Engine` 是唯一部署单元（`http.Handler` 接口实现），`ginS/` 为全局单例示例服务器，非库代码。
- **internal 边界**：`routergroup.go` 仅依赖 `internal/bytesconv`（在 tree.go 中用于零拷贝字节转换），无越界 import；接口 `IRoutes`/`IRouter` 定义在消费侧（用户代码）与实现侧（RouterGroup/Engine）之间，体现依赖倒置。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 路由组与注册架构图 | `routergroup-architecture.html` | architecture | showcase |
| 单条路由注册调用时序 | `routergroup-sequence.html` | sequence | showcase |

JSON IR 位于 `json/` 目录（`routergroup-architecture.json`、`routergroup-sequence.json`）。
本叶子未生成 workflow / dataflow / lifecycle 图：路由注册是一次性「拼路径+拼链+写树」的同步调用链，无多角色泳道流程、无数据管道、无单实体状态机；其组件拓扑与调用时序已由 architecture + sequence 完整表达，按资源节省原则省略。
