# gin 系统总览（system-overview）

> 源码基准：`github.com/gin-gonic/gin` master @ `5c6a15f`（Go 1.26.0，MIT，98 个 Go 文件 / 约 2.4 万行含测试）。
> 分析方式：archify 按"系统级 → 域 → 叶子"两级拆分——6 个域（routing / handling / middleware / binding / render / support）、15 个叶子，每个叶子为设计文档级分析单元。

## 1. 功能总览

gin 是一个高性能 HTTP Web 框架（Martini-like API，httprouter 血缘，零分配路由），核心能力：

| 能力 | 实现要点 | 归属 |
|---|---|---|
| 零分配基数树路由 | 前缀共享 radix tree，`:param`/`*wildcard` 通配匹配、tsr 尾斜杠重定向 | routing |
| 路由分组与注册 | RouterGroup 前缀组、中间件继承、Method 快捷方法、QUERY 方法（RFC 10008） | routing |
| 请求生命周期 | Engine.ServeHTTP → handleHTTPRequest → 路由匹配 → Context 处理链 → 响应 | handling |
| 中间件链式执行 | Use 注册 + combineHandlers 组合 + c.Next/c.Abort 游标协议 | middleware |
| 多格式绑定 | JSON/XML/YAML/TOML/MsgPack/Protobuf/BSON/Plain/Form/Query/Header + 结构校验 | binding |
| 多格式渲染 | 同格式集 + Text/HTML/Redirect/Data/Reader/PDF | render |
| 集中错误管理 | Error/ErrorType 聚合、ByType 过滤、JSON 序列化 | support |
| 可插拔 JSON codec | codec/json 接口 + 标准库/go-json/jsoniter/sonic 四实现 | support |
| 内置中间件 | Recovery 防崩溃、Logger、BasicAuth、静态文件服务 | middleware |
| 多运行方式 | Run/RunTLS/RunFd/RunListener/RunUnix/RunQUIC | handling |

## 2. 解决的问题

- **路由性能**：基数树将路径匹配复杂度压缩到路径长度量级，注册期一次性构建、运行期零分配只读匹配（`tree.go:418 getValue`）。
- **中间件组合复杂性**：把"全局中间件 → 组中间件 → 路由处理器"的合并固化到注册期（`routergroup.go:251 combineHandlers`），请求期仅靠 `c.Next()` 游标线性执行（`context.go:198`），链长超限即 panic 防护。
- **绑定与渲染的格式爆炸**：以 `Binding`/`Render` 两个接口统一 10+ 种格式，`Context` 提供 `Bind`/`ShouldBind` 与 `JSON`/`HTML`/`Stream` 等快捷入口（`context.go:780`、`context.go:1260`），业务代码与具体编解码库解耦。
- **错误与崩溃韧性**：集中错误聚合（`errors.go:98 ByType`）+ Recovery 中间件把 panic 转 500（`recovery.go:53`），避免单请求崩溃拖垮进程。
- **性能敏感路径**：`internal/bytesconv` 零拷贝字符串转换、`sync.Pool` 上下文池复用（`gin.go:662 ServeHTTP`）、可插拔高性能 JSON codec 各取所需。

## 3. 系统边界

| 边界 | 内容 | 说明 |
|---|---|---|
| 上边界 | HTTP 客户端 | 请求方，不在本仓库源码内 |
| 下边界 | net/http（Server/ResponseWriter）、go-playground/validator、sonic/jsoniter/go-json、quic-go、文件系统 | 标准库与第三方依赖，不在本仓库源码内 |
| 内边界 | 顶层单包 `gin`（engine/context/tree/routergroup/...）+ binding/ + render/ + internal/ + codec/json | 本仓库全部源码，按 6 域归组 |
| 侧边界 | `ginS/` 示例服务器（gins.go + README） | 示例而非库代码，仅作运行方式参考 |

**不做什么**：不内置模板引擎（HTML 渲染需用户配置 `LoadHTMLGlob` 等）；不提供数据库/ORM 访问；不管理进程生命周期与日志文件（Logger 中间件仅向 writer 输出）；不约束业务分层（路由处理器即普通函数签名 `func(*Context)`）。

## 4. 图表说明

| 图 | 表达的核心语义 | 质量档位 | 说明 |
|---|---|---|---|
| [system-architecture.html](system-architecture.html) | 组件分层拓扑：外部依赖（client/net/http）与 gin 核心 11 组件的静态关系 | standard | 组件多、跨层连接复杂，系统级允许 standard；render 成功 |
| [system-sequence.html](system-sequence.html) | 请求生命周期：client→net/http→Engine→路由树→中间件链→业务处理器→响应写入→返回 | showcase | 7 参与者、10 条消息，一次通过 |
| [system-dataflow.html](system-dataflow.html) | 请求数据流：入口→路由→参数解析→业务处理→响应输出，含绑定失败/panic→错误聚合→JSON 错误响应分支 | standard | 多分支跨列通道，系统级允许 standard；render 成功 |

> 质量档位口径：叶子级图目标 showcase（合格线），系统级图允许 standard（方案 5）。standard 仅表示布局约束较 showcase 宽松，内容与交互性不受影响（HTML 均为 700KB+ 自包含交互式）。
> 图 JSON IR 源文件见 `json/` 目录（与 HTML 同名）。
