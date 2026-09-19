# 基础设施（infra）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` commit `5c6a15f`。

## 1. 域职责

本域承载 Gin 框架的横切基础设施：错误收集模型、运行时模式与全局输出、全局单例门面、以及 internal 包内的零拷贝与文件系统辅助。它们不直接处理路由，但被核心引擎、中间件、绑定/渲染各层广泛依赖。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| error-model | [error-model.md](error-model/error-model.md) | [架构图](error-model/error-model-architecture.html) | — | Error/ErrorType 位标志 + errorMsgs 错误链，JSON 序列化 |
| runtime-mode | [runtime-mode.md](runtime-mode/runtime-mode.md) | [架构图](runtime-mode/runtime-mode-architecture.html) | — | debug/release/test 三态原子切换 + DefaultWriter/ErrorWriter |
| global-singleton | [global-singleton.md](global-singleton/global-singleton.md) | [架构图](global-singleton/global-singleton-architecture.html) | — | ginS 包级函数群透明转发到 sync.OnceValue 全局 Engine |
| internal-utils | [internal-utils.md](internal-utils/internal-utils.md) | [架构图](internal-utils/internal-utils-architecture.html) | — | bytesconv 零拷贝 + internal/fs 适配 + 静态 FileSystem |

## 3. 域级机制细节

- **全局状态并发安全**：`ginMode int32`（runtime-mode）与 `engine sync.OnceValue`（global-singleton）都用原子原语，保证路由注册期与请求期并发访问无数据竞争。
- **错误流经全栈**：`c.Error`（context.go:262）把错误喂入 error-model 的 `c.Errors`，再被 Logger/Recovery 消费；错误类型位（Private/Public/Bind/Render）决定其可见范围。
- **internal 边界**：`internal/bytesconv`、`internal/fs` 受 Go 编译器访问控制，零拷贝工具不泄漏给外部项目；`fs.go` 的 `Dir/OnlyFilesFS` 在根包供 routergroup 静态文件使用。
- **门面与实例并存**：`ginS`（global-singleton）是"单服务"语法糖，`gin.Default()`/`gin.New()` 是标准用法；两者共享同一份包级模式与 Writer。

## 4. 域级图

本域不单独绘制域级架构图：四个叶子分别覆盖错误、模式、门面、工具四个正交切面，其关系已在系统级架构图与各叶子图中表达。
