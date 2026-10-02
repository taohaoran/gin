# gin 架构分析文档（archify）

> 基于 `github.com/gin-gonic/gin` master @ `5c6a15f`（Go 1.26.0，MIT）的深度源码架构分析。
> 由 archify 工具链生成：设计文档级简体中文 MD + 交互式 HTML 图（Architecture / Sequence / DataFlow / Workflow / Lifecycle 五种图型按语义选型）。
> 拆分口径：系统级 → 域 → 叶子两级拆分，共 **6 个域、15 个叶子**。

## 系统级

| 文档 / 图 | 说明 | 质量档位 |
|---|---|---|
| [系统总览](system-overview.md) | 功能总览 / 解决的问题 / 系统边界 / 图表说明 | — |
| [系统架构图](system-architecture.html) | 组件分层与边界拓扑 | standard |
| [请求生命周期时序图](system-sequence.html) | 请求从入口到响应的调用时序 | showcase |
| [请求数据流图](system-dataflow.html) | 请求解析 / 处理 / 响应与错误路径 | standard |

## 域与叶子索引

| 域 | 域总览 | 叶子 |
|---|---|---|
| routing（路由） | [routing.md](routing/routing.md) | [routergroup](routing/routergroup/routergroup.md) · [tree](routing/tree/tree.md) · [path](routing/path/path.md) |
| handling（请求处理） | [handling.md](handling/handling.md) | [engine](handling/engine/engine.md) · [context](handling/context/context.md) · [response-writer](handling/response-writer/response-writer.md) |
| middleware（中间件） | [middleware.md](middleware/middleware.md) | [middleware-core](middleware/middleware-core/middleware-core.md) · [logger](middleware/logger/logger.md) · [recovery-auth](middleware/recovery-auth/recovery-auth.md) |
| binding（绑定） | [binding.md](binding/binding.md) | [binding-core](binding/binding-core/binding-core.md) · [binding-formats](binding/binding-formats/binding-formats.md) |
| render（渲染） | [render.md](render/render.md) | [render-core](render/render-core/render-core.md) · [render-formats](render/render-formats/render-formats.md) |
| support（支撑） | [support.md](support/support.md) | [errors](support/errors/errors.md) · [internal-utils](support/internal-utils/internal-utils.md) |

每个叶子目录内含：设计文档级 MD（10 小节，含真实 `文件:函数:行号` 引用）、≥2 张交互式 HTML 图、同名 JSON IR（`json/` 子目录）。

## 文档与图约定

- **语言**：所有 MD 与图中作者内容一律简体中文；代码标识符、源码路径、flag 名保持原文。文件名/目录名英文短横线。
- **质量档位**：叶子级图目标 showcase（合格线）；系统级图允许 standard（组件多、跨层连接复杂为预期内）。任何落 standard 的图均在对应 MD 第 10 节披露失败检查名与修复动作。
- **语言适配口径**：主语言 Go。并发模型为"每请求一 goroutine + sync.Pool 上下文池"；路由树注册期单线程构建、运行期多 goroutine 只读匹配；非 K8s Reconciler/informer 模式（差异已在各叶子第 9 节说明）。
- **外部依赖**：net/http、go-playground/validator、各编解码库（sonic/jsoniter/go-json）、文件系统等均标注"不在本仓库源码内"（图内 `external` 类型 + MD 第 7 节）。
