# support（支撑设施）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 域职责

support 域收拢 gin 被各业务域（路由、中间件、render、binding）复用的**底层支撑代码**：一是类型化错误聚合 `errors.go`（`c.Errors` 的收集/分类/序列化），二是散落各处的工具函数与可插拔 JSON 编解码 `codec/json`。本域是**被依赖的底层**，不反向依赖业务域；它提供错误收纳箱与高性能可替换的 JSON 后端，是框架可观测性与序列化一致性的基础。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 其他图 | 职责一句话 |
|------|------|--------|--------|--------|-----------|
| 域总览 | 本文 | [架构图](support-architecture.html) | [时序图](support-sequence.html) | [数据流图](support-dataflow.html) | 错误聚合 + 工具/JSON codec 总览 |
| errors | [errors.md](errors/errors.md) | [架构图](errors/errors-architecture.html) | — | [数据流图](errors/errors-dataflow.html) | ErrorType 位掩码分类、errorMsgs 聚合与序列化 |
| internal-utils | [internal-utils.md](internal-utils/internal-utils.md) | [架构图](internal-utils/internal-utils-architecture.html) | — | [数据流图](internal-utils/internal-utils-dataflow.html) | 零拷贝转换、路径辅助、可插拔 JSON codec 四后端 |

## 3. 域级机制细节

- **错误位掩码分类**：`ErrorType` 是 `uint64` 位标志（Bind/Render/Public/Private/Any），一个错误可同时打多标签；`IsType` 位与查询 O(1)。未标注的错误经 `c.Error` 默认归 `ErrorTypePrivate`，只进日志不进响应。
- **错误与中间件协作**：Recovery（`c.Error` 推进 panic）→ Logger（读 `ByType(ErrorTypePrivate)` 打印）→ ErrorLoggerT（按类型 JSON 输出），三者经 `c.Errors` 解耦（详见 middleware 域）。
- **JSON codec 可插拔矩阵**：`codec/json/api.go` 定义 `Core` 接口与包级 `var API`；四个实现文件靠 `//go:build` tag 在编译期四选一（标准库缺省 / go-json / jsoniter / sonic），`init()` 把自身赋给 `API`。业务代码面向接口，无感知切换后端。
- **internal 隔离**：`internal/bytesconv`（unsafe 零拷贝）与 `internal/fs` 受 Go internal 机制保护，仅仓库内可用；被 recovery/auth 等文件正常引用。

## 4. 域级图

![support 域架构图](support-architecture.html)

![错误序列化调用时序](support-sequence.html)

![support 域数据管道](support-dataflow.html)

**省略说明**：本域未生成 workflow/lifecycle 图——支撑设施是无状态纯函数调用与编译期装配（已由 architecture 拓扑 + dataflow 管道 + sequence 调用链表达），无多角色泳道流程、也无单一实体运行时状态机，按资源节省原则省略。
