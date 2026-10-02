# routing 域（路由与匹配）总览

> 本域包含路由组、基数树、路径规范化三个叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 域职责

routing 域负责把「用户声明的路由」转换为「请求期可零分配查找的基数树结构」，并在请求到来时把 URL 路径映射到一条中间件+处理函数链。它横跨两个阶段：

- **注册期（启动前，单线程）**：`RouterGroup` 按路径前缀与中间件分组，合并出完整 `HandlersChain`，经 `Engine.addRoute` 写入按 HTTP 方法分棵的基数树；冲突即 panic。
- **运行期（监听后，并发只读）**：`Engine.handleHTTPRequest` 取出请求路径，按需经 `cleanPath` 规范化，在对应方法树上 `getValue` 静态优先、通配兜底地匹配，提取 `Params`，把 `handlers` 交给 `Context` 执行；未命中则按 `tsr` 做尾斜杠重定向 / 405 / 404。

域核心代码路径：`routergroup.go`（269 行）、`tree.go`（950 行，httprouter 血缘）、`path.go`（203 行，标准库 path 血缘），以及 `gin.go` 中 `Engine.addRoute`（`gin.go:364`）与 `Engine.handleHTTPRequest`（`gin.go:690`）两个衔接点。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| 域总览 | 本文 | [域架构图](routing-architecture.html) | [匹配时序](routing-sequence.html) · [匹配数据流](routing-dataflow.html) | 路由注册与匹配总入口 |
| routergroup | [routergroup.md](routergroup/routergroup.md) | [架构图](routergroup/routergroup-architecture.html) | [注册时序](routergroup/routergroup-sequence.html) | 路由组嵌套、中间件合并、方法快捷注册 |
| tree | [tree/tree.md](tree/tree.md) | [架构图](tree/tree-architecture.html) | [匹配数据流](tree/tree-dataflow.html) | 基数树插入与零分配匹配、参数提取、tsr |
| path | [path/path.md](path/path.md) | [架构图](path/path-architecture.html) | [规范化数据流](path/path-dataflow.html) | cleanPath 路径规范化与去重复斜杠 |

## 3. 域级机制细节

- **按方法分棵**：`Engine.trees` 是 `methodTrees` 切片（`tree.go:50`），每个 HTTP 方法一棵独立基数树；请求期先线性定位方法树再 `getValue`。405 判定时会跨方法树再查一遍（`gin.go:746`）。
- **静态优先、通配兜底**：基数树中静态子节点按 `indices` 索引快速定位，通配子节点（`:param`/`*catchAll`）恒排在 `children` 末尾（`tree.go:71`）；走静态分支前用 `skippedNodes` 快照当前节点，走错可回退（`tree.go:460`）。
- **零分配**：注册期一次性构建不可变树；运行期 `getValue` 复用 `Context` 上的 `Params`/`skippedNodes` 切片容量（`tree.go:500`），路径用字符串切片共享，`cleanPath`/`bufApp` 在未改路径时返回子串。
- **失败即 panic（注册期）**：方法名非法、路径冲突、重复注册、通配语法错误全部启动期 panic，运行期只返回 `handlers==nil + tsr` 标志，由引擎决定重定向/兜底。
- **与下游 context 域的接缝**：routing 域只负责「产出 `handlers HandlersChain` 与 `Params`」，中间件链的顺序执行（`Context.Next`/`index` 推进）由 context 域负责。

## 4. 域级图

![routing 域整体架构图](routing-architecture.html)

![请求期路由匹配时序](routing-sequence.html)

![请求路径匹配数据流](routing-dataflow.html)

> 域级图质量：`routing-architecture.html`、`routing-sequence.html` 为 showcase；`routing-dataflow.html` 落 standard（失败检查 `composition/proper-crossing`：通配分支 `n3→n4` 与静态命中 `n2→n5` 两条跨列边在通道交叉，伴 `label-route-clearance` 标签间距 3px；已加宽 viewBox、`n2→n5` 改 bottom-channel、删除穿节点的兜底边，两轮后按协议落 standard）。
