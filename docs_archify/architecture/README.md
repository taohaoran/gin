# gin Web 框架架构分析

> 本目录由 archify 对 [gin](https://github.com/gin-gonic/gin) 源码深度分析生成，全部内容为简体中文。
> gin 是一个用 Go 编写的高性能 Web 框架（路由树 + 中间件模型），以库形态提供 `net/http` 之上的路由、上下文、请求绑定与响应渲染能力。

## 系统级入口

| 文档 / 图 | 说明 |
|---|---|
| [系统总览](system-overview.md) | 功能总览、解决的问题、系统边界、不做什么 |
| [系统架构图](system-architecture.html) | Engine / RouterGroup / 路由树 / Context / 绑定渲染 / 中间件分层 |
| [请求生命周期时序图](system-request-sequence.html) | net/http → ServeHTTP → 路由匹配 → 中间件链 → 业务 handler → 写回 |
| [请求处理数据流](system-dataflow.html) | 接入 → 上下文/路由 → 中间件链/业务 handler → 渲染写回 |

## 域与叶子索引（7 域 / 21 叶子）

### 核心引擎 `core-engine/`
域总览：[core-engine.md](core-engine/core-engine.md)

- [engine-lifecycle](core-engine/engine-lifecycle/engine-lifecycle.md) — Engine 结构体、New/Default/With、Run*/ServeHTTP、可信代理 CIDR、Context 池化（架构图 + 请求时序图）
- [context-object](core-engine/context-object/context-object.md) — Context 结构体、Next/Abort 流控、Keys、绑定/渲染/取值方法族、context.Context 适配
- [router-group](core-engine/router-group/router-group.md) — RouterGroup、IRouter/IRoutes、group 嵌套、combineHandlers、静态文件挂载
- [response-writer](core-engine/response-writer/response-writer.md) — responseWriter 拦截层、ResponseWriter 接口、OnlyFilesFS/Dir

### 路由树 `router-tree/`
域总览：[router-tree.md](router-tree/router-tree.md)

- [radix-tree](router-tree/radix-tree/radix-tree.md) — node 四类型、addRoute/getValue、通配符冲突、skippedNode 回滚、大小写不敏感查找（架构图 + 数据流图）
- [path-utils](router-tree/path-utils/path-utils.md) — cleanPath、joinPaths、resolveAddress、assert1、nameOfFunction

### 请求绑定 `request-binding/`
域总览：[request-binding.md](request-binding/request-binding.md)

- [binding-registry](request-binding/binding-registry/binding-registry.md) — Binding 系列接口、Default() 分派、validator v10 封装
- [body-bindings](request-binding/body-bindings/body-bindings.md) — json/xml/yaml/toml/protobuf/msgpack/bson/plain 各 Body 绑定器
- [form-bindings](request-binding/form-bindings/form-bindings.md) — form/query/uri/header 绑定、tag 反射映射、multipart 阈值

### 响应渲染 `response-render/`
域总览：[response-render.md](response-render/response-render.md)

- [render-interface](response-render/render-interface/render-interface.md) — Render 接口、两阶段写、JSON 各变体（架构图 + 时序图）
- [markup-renderers](response-render/markup-renderers/markup-renderers.md) — xml/yaml/toml/protobuf/msgpack/bson/data/text/pdf
- [html-redirect-reader](response-render/html-redirect-reader/html-redirect-reader.md) — HTML Debug/Production 双轨模板、Redirect、Reader 流式透传

### JSON 抽象 `codec-json/`
域总览：[codec-json.md](codec-json/codec-json.md)

- [json-backend-abstraction](codec-json/json-backend-abstraction/json-backend-abstraction.md) — codec/json 统一 API、go-json/jsoniter/sonic build tag 编译期选择

### 内置中间件 `builtin-middleware/`
域总览：[builtin-middleware.md](builtin-middleware/builtin-middleware.md)

- [logger-middleware](builtin-middleware/logger-middleware/logger-middleware.md) — Logger 中间件、日志字段、跳过路径
- [recovery-middleware](builtin-middleware/recovery-middleware/recovery-middleware.md) — panic 恢复、堆栈打印、Abort 500（架构图 + 时序图）
- [basic-auth](builtin-middleware/basic-auth/basic-auth.md) — BasicAuth、恒定时间比较、用户注入
- [debug-print](builtin-middleware/debug-print/debug-print.md) — DebugPrintRoute 调试输出

### 基础设施 `infra/`
域总览：[infra.md](infra/infra.md)

- [error-model](infra/error-model/error-model.md) — Error 结构、TypeError、ErrorManager 错误链
- [runtime-mode](infra/runtime-mode/runtime-mode.md) — Debug/Release/Test 模式、全局 Writer、deprecated 迁移
- [global-singleton](infra/global-singleton/global-singleton.md) — 包级默认引擎、ginS 透明转发
- [internal-utils](infra/internal-utils/internal-utils.md) — internal/bytesconv 零拷贝、internal/fs、internal 边界

## 质量说明

- 全部 archify 图均 render 退出码 0、自包含 HTML（约 800KB）。
- 质量档位：多数叶子图为 **showcase**；少量多组件图（context-object、router-group、radix-tree、runtime-mode 及系统架构/数据流）因布局校验降为 **standard**，原因已在各叶子 MD 第 10 节与系统图中如实披露。
- 第三方依赖（validator、go-json、jsoniter、sonic、protobuf、msgpack、bson、yaml、toml 等）均标注"不在本仓库源码内"。
