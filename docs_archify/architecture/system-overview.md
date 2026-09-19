# gin 系统总览

## 一、功能总览

gin 是一个 Go 语言编写的 Web 框架，在标准库 `net/http` 之上提供：

- **路由**：基于基数树（radix tree）的高性能路由，支持静态段、参数段（`:name`）、通配段（`*filepath`）、路由分组与大小写不敏感查找。
- **中间件**：洋葱模型的 handler 链，`Context.Next()` 控制流推进，内置 Logger / Recovery / BasicAuth 等。
- **上下文**：每请求一个 `Context`，承担路径参数、请求取值、请求绑定、响应渲染、错误收集与 `context.Context` 适配。
- **请求绑定**：按 Content-Type 分派到 JSON/XML/YAML/TOML/表单/查询串/URI/Header 绑定器，并接入 `go-playground/validator` 校验。
- **响应渲染**：统一 `Render` 接口，覆盖 JSON（含 Indented/Secure/JSONP/Ascii/Pure 变体）、XML/YAML/TOML/Protobuf/HTML/Text/Data/PDF/Redirect/Reader。
- **JSON 后端抽象**：`codec/json` 通过 build tag 在 encoding/json、jsoniter、字节 sonic 之间编译期切换。
- **工程设施**：运行模式（Debug/Release/Test）、错误模型、包级单例 `ginS`、`internal/bytesconv` 零拷贝工具。

## 二、解决的问题

1. **路由匹配性能与冲突处理**：用基数树把 URL 模式压缩成单棵前缀树，注册时校验静态/参数/通配段冲突，请求时 O(URL) 匹配。
2. **每请求对象复用**：`Context` 用 `sync.Pool` 池化复用，`reset()` 清理状态，避免高并发下的堆分配。
3. **请求/响应编码统一**：绑定与渲染两套接口把"读请求体 → 结构体"和"结构体 → 响应体 + Content-Type"标准化，业务只面对 `Context`。
4. **中间件可组合**：`RouterGroup.Use` 把中间件与业务 handler 合并成有序链，支持分组与嵌套。
5. **JSON 序列化可替换**：用 build tag 把 JSON 后端编译期钉死，运行时零分支，兼顾默认兼容性与高性能后端。

## 三、系统边界

| 方向 | 边界对象 | 说明 |
|---|---|---|
| 上 | `net/http`（Go 标准库） | gin 实现 `http.Handler` 接口，由标准库 Server 驱动；不在本仓库源码内 |
| 下 | 业务 Handler（用户代码） | 路由命中后最终调用的用户 handler；不在本仓库源码内 |
| 内 | 路由树 + Context + binding/render + 中间件 | 本仓库全部核心源码 |
| 侧 | validator / JSON 后端 / 模板引擎 | 第三方依赖，均不在本仓库源码内 |

**不做什么**：
- 不实现 HTTP 服务器本身（依赖 `net/http`，仅提供 `Run` 系列便捷启动）。
- 不做服务发现、配置中心、RPC 框架、ORM、数据库连接池。
- 不自带 WebSocket 服务端实现（仅透传 `Hijacker` 接口）。
- 不做请求级重试/熔断/限流（由中间件生态或用户自行实现）。

## 四、核心视图

- [系统架构图](system-architecture.html)：分层与依赖关系。
- [请求生命周期时序图](system-request-sequence.html)：从 `ServeHTTP` 到响应写回的主路径。
- [请求处理数据流](system-dataflow.html)：接入 → 路由 → 中间件链 → 业务 handler → 渲染写回。

## 五、Go 语言适配口径

主语言为 Go 库形态（无 `cmd/` 多二进制、无 main 包）。分析覆盖：

- **并发模型**：`Context` 的 `sync.Pool` 复用与 `reset()`、`defaultValidator` 的 `sync.Once` 懒初始化、`atomic` 状态切换、`context.Context` 通过 `c.Request.WithContext` 传播。
- **internal 边界**：`internal/bytesconv`、`internal/fs` 仅 gin 包内可导入，对外不可见。
- **单二进制库形态**：通过 `Engine` 单实例 + 包级 `ginS` 全局转发提供两种使用范式。
- **不做**：K8s 控制器模式 / informer / reconciler 分析（本项目非 K8s 平台类）。
