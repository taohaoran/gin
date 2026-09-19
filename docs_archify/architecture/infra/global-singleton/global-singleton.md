# 全局单例门面 ginS（global-singleton）

> 本文是 `infra` 域下的叶子子系统文档。域级总览见 `../infra.md`，
> 本文只展开 `ginS` 包级门面如何代理到全局 `*gin.Engine`，不重复 Engine 本身生命周期（见 engine-lifecycle 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `ginS/gins.go`（160 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 全局引擎懒初始化 | `engine = sync.OnceValue(gin.Default())` | `gins.go:15` |
| 路由方法转发 | `GET/POST/PUT/PATCH/DELETE/HEAD/OPTIONS/Any/Handle` | `gins.go:56-98` |
| 分组与中间件 | `Group/Use/NoRoute/NoMethod` | `gins.go:40-53`、`gins.go:124` |
| 静态资源 | `Static/StaticFS/StaticFile` | `gins.go:100-119` |
| HTML 模板 | `LoadHTMLGlob/LoadHTMLFiles/LoadHTMLFS/SetHTMLTemplate` | `gins.go:20-37` |
| 启动服务 | `Run/RunTLS/RunUnix/RunFd` | `gins.go:136-159` |
| 路由表查询 | `Routes()` | `gins.go:129` |

对外暴露点：`ginS.GET("/", ...)`、`ginS.Run(":8080")` 等包级函数，免去用户持有 `*gin.Engine` 变量。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `engine` | `gins.go:15` | `sync.OnceValue[*gin.Engine]`，返回全局唯一引擎 |
| 包级函数群 | `gins.go:20-159` | 每个函数体仅一行 `engine().X(...)` |

设计要点：`sync.OnceValue`（Go 1.21+）保证并发首次调用时只执行一次 `gin.Default()`，后续调用直接返回缓存实例；函数体零业务逻辑，纯转发。

## 3. 关键调用链

**主路径：用户包级调用 → 全局引擎（以 `GET` 为例）**

1. 用户调用 `ginS.GET("/path", handler)`（`gins.go:66`）。
2. 函数体 `return engine().GET(relativePath, handlers...)`（`gins.go:67`）。
3. `engine()` 首次调用时触发 `sync.OnceValue` 的初始化函数 `gin.Default()`（`gins.go:15-17`）——`Default()` 在 `gin.go:236` 内 `New()` 后 `Use(Logger(), Recovery())`（`gin.go:238-239`）。
4. 后续所有包级调用直接复用该同一实例。

**启动服务（`Run`，`gins.go:136`）**

- `return engine().Run(addr...)`（`gins.go:137`），阻塞当前 goroutine 直到出错；底层仍包 `http.Server`（engine-lifecycle 叶子）。

**中间件挂载（`Use`，`gins.go:124`）**

- `return engine().Use(middlewares...)`（`gins.go:125`），与持有实例时的 `r.Use(...)` 语义完全一致。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| 初始引擎 | `gin.Default()`（已挂 Logger+Recovery） | `gins.go:16` |
| 惰性初始化 | 首次任意包级调用才创建 Engine，而非包加载时 | `gins.go:15` |
| 多启动方式 | `Run/RunTLS/RunUnix/RunFd` 全量透传 | `gins.go:136-159` |

## 5. 错误与重试语义

- 门面层不包装错误：`Run*` 的 error 原样返回（`gins.go:137`），`Handle/Group` 等注册类方法返回值原样透传。
- `sync.OnceValue` 的初始化函数若 panic，panic 会在首次调用处抛出；之后调用方持有已损坏的全局状态，无法重置。
- 无重试。

## 6. 并发细节

- `sync.OnceValue` 内部用 `sync.Once` + 互斥，保证多 goroutine 并发首次调用 `engine()` 时只执行一次初始化（`gins.go:15`）。
- `engine()` 返回后，对 `*gin.Engine` 的并发访问遵循 Gin 自身约定：路由注册应在服务启动前完成，运行期只读路由树。
- 门面函数本身无锁，热路径直接转发，零额外开销。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `gins.go` 全部：全局单例门面的转发函数群。

**Out-of-Scope（不在本仓库源码内）**
- `gin.Default/New`、`Engine` 全部方法实现在 `gin.go`/`routergroup.go`（engine-lifecycle / routergroup 叶子）。
- `html/template`、`net/http` 标准库：模板与 HTTP，外部依赖。
- 本叶子不实现任何业务逻辑，纯粹是语法糖门面。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户业务代码直接调用 `ginS.XXX`。
- 本叶子 → 下游：所有调用转发到 `*gin.Engine`（engine-lifecycle 叶子）；初始化走 `gin.Default()`，自动带上 builtin-middleware 域的 Logger 与 Recovery。
- 本叶子 → 相邻：与 runtime-mode 共享同一份包级模式状态。

## 9. 语言专项适配口径

- **Go 并发模型**：`sync.OnceValue` 是 Go 1.21 的类型化单次执行原语，替代手写 `var once sync.Once; var engine *Engine` 的样板，是本叶子的核心并发保证。
- **门面模式**：包级函数群是面向"全局只有一个服务"的简化用法；多服务场景应改用 `gin.New()`/`gin.Default()` 持有实例（engine-lifecycle 叶子）。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| ginS 门面架构图 | `global-singleton-architecture.html` | architecture | showcase |
| JSON IR | `json/global-singleton-architecture.json` | — | — |

本叶子不补时序图：调用链为"包级函数→engine()→Default()"的单层转发，架构图足以表达。
