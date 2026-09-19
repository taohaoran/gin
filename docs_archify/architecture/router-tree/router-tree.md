# 路由树（router-tree）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin`（Go 1.26 指令），commit `5c6a15f`。tree.go 算法源自 julienschmidt/httprouter（BSD-3）。

## 1. 域职责

router-tree 是 gin 的"路由匹配引擎"：以基数树（radix tree）按 HTTP method 分树存储路由，注册期把路径插入树并按访问热度重排，运行期把请求路径匹配到 handler 链与路径参数。本域覆盖路由树本体与路径工具函数两部分。

核心代码路径：
- 路由树：`tree.go`（`node`/`addRoute`/`getValue`/`findCaseInsensitivePath`）
- 路径工具：`path.go`（`cleanPath`/`removeRepeatedChar`）+ `utils.go`（`joinPaths`/`resolveAddress`/`assert1` 等）

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| radix-tree | [radix-tree.md](radix-tree/radix-tree.md) | [架构图](radix-tree/radix-tree-architecture.html) | [数据流图](radix-tree/radix-tree-dataflow.html) | 四种节点类型、插入/匹配、wildcard 冲突 panic、skippedNode 回滚、大小写不敏感查找 |
| path-utils | [path-utils.md](path-utils/path-utils.md) | [架构图](path-utils/path-utils-architecture.html) | — | cleanPath/removeRepeatedChar/joinPaths/resolveAddress/assert1/nameOfFunction |

## 3. 域级机制细节

- **按 method 分树**：`Engine.trees` 是 `methodTrees`（`tree.go:50`），每个 HTTP method 一棵 `root node`；`handleHTTPRequest` 线性遍历找到同 method 根后调 `root.getValue`。
- **节点类型与约束**：`nodeType` 分 `static/root/param(:)/catchAll(*)`（`tree.go:90-97`）；每个父节点的 `children` 末尾至多一个 wildcard 子节点（`wildChild` 标记）。catchAll 只允许在路径末尾，注册期冲突直接 panic。
- **优先级热点前置**：`incrementChildPrio`（`tree.go:111`）把高频访问子节点向前移并同步重排 `indices` 字符序，使热路径落在 indices 前端。
- **skippedNode 回滚替代回溯**：`getValue` 在 `wildChild` 节点先把当前节点快照压入 `skippedNodes` 栈（`tree.go:434-448`），静态子节点走错路时弹栈恢复 path/node/paramsCount 重试（`tree.go:460-472`），避免传统回溯的重复遍历。
- **复用切片零分配**：`getValue` 复用 `Context.params`/`Context.skippedNodes`（由 `Engine.allocateContext` 预分配容量），不在请求热路径上分配。
- **路径工具的零分配设计**：`cleanPath`（`path.go:23`）用 `r/w` 双下标单遍扫描 + `bufApp`（`path.go:128`）惰性缓冲，常见无需清理的路径直接返回原串子串，零分配。

## 4. 与相邻域的边界

- 上游 `core-engine` 域：`Engine.addRoute`（注册期）与 `handleHTTPRequest`（运行期）调用本域的 `addRoute`/`getValue`/`findCaseInsensitivePath`。
- 本域不执行 handler 链（core-engine 的 `Context.Next`）、不做重定向（core-engine 的 `redirectRequest`）。
