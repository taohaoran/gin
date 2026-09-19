# 基数路由树（radix-tree）

> 本文是 `router-tree` 域下的叶子子系统文档。域级总览见 `../router-tree.md`。
> 本文只展开 **路由树的数据结构、插入（addRoute/insertChild）、匹配（getValue）、wildcard 冲突校验、skippedNode 回滚与大小写不敏感查找**，不重复展开 Engine 如何选 method 树与派发（见 `../core-engine/engine-lifecycle/engine-lifecycle.md`）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。主源码文件：`tree.go`（源自 julienschmidt/httprouter，BSD-3）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 路由参数模型 | `Param{Key,Value}` / `Params`，`Get/ByName` 取值 | `tree.go:17`、`tree.go:25`、`tree.go:29`、`tree.go:40` |
| 按 method 分树 | `methodTree{method,root}` / `methodTrees`，`get(method)` 线性找根 | `tree.go:45`、`tree.go:50`、`tree.go:52` |
| 四种节点类型 | `static/root/param(:)/catchAll(*)` | `tree.go:90`–`tree.go:97` |
| node 结构 | `path/indices/wildChild/nType/priority/children/handlers/fullPath` | `tree.go:99` |
| 最长公共前缀 | `longestCommonPrefix` 决定边切分点 | `tree.go:61` |
| 子节点插入与排序 | `addChild`（wildcard 恒置末尾）、`incrementChildPrio`（按 priority 热点前置） | `tree.go:71`、`tree.go:111` |
| 路由插入 | `addRoute`：空树/切边/静态子节点/wildcard 冲突 panic/挂 handler | `tree.go:135` |
| wildcard 解析 | `findWildcard`：找 `:param/*` 段，校验非法字符与转义 | `tree.go:253` |
| 递归建子树 | `insertChild`：param/catchAll 分支、catchAll 仅允许在末尾 | `tree.go:288` |
| 路由匹配 | `getValue`：静态 indices 匹配 → param/catchAll 取值 → skippedNode 回滚 | `tree.go:418` |
| 匹配返回值 | `nodeValue{handlers,params,tsr,fullPath}` | `tree.go:400` |
| 回滚栈 | `skippedNode{path,node,paramsCount}`，param 与 static 冲突兜底 | `tree.go:407` |
| 大小写不敏感 | `findCaseInsensitivePath`/`findCaseInsensitivePathRec`，处理 Unicode rune 4 字节缓存 | `tree.go:671`、`tree.go:703` |
| 统计容量 | `countParams`（`:`+`*` 数）/`countSections`（`/` 数） | `tree.go:80`、`tree.go:86` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `nodeType uint8` | `tree.go:90` | 节点类型枚举（static/root/param/catchAll） |
| `node` struct | `tree.go:99` | 基数树节点；`children` 末尾至多一个 wildcard 子节点（`wildChild` 标记） |
| `methodTrees` | `tree.go:50` | 每个 HTTP method 一棵树的切片 |
| `nodeValue` | `tree.go:400` | `getValue` 的四元返回值（含 tsr 尾斜杠建议） |
| `skippedNode` | `tree.go:407` | 回滚栈元素，记录走过 static 子节点前的快照 |
| `Params` | `tree.go:25` | 有序参数切片，`c.Params` 的底层类型 |
| `longestCommonPrefix` | `tree.go:61` | 插入期切分共享前缀边的工具函数 |

## 3. 关键调用链

**链路 A：插入一条路由（addRoute）**

1. 空树时 `insertChild(path,...)` 并把根 `nType=root`（`tree.go:140-143`）。
2. 非空时 `longestCommonPrefix(path, n.path)` 求公共前缀 `i`；若 `i < len(n.path)` 说明需切边——把旧节点剩余路径拆成一个新 static 子节点，当前节点收窄为前缀（`tree.go:153-175`）。
3. 余下 `path` 首字节 `c`：若 `n.nType==param` 且 `c=='/'` 且单子节点，下沉进 param 子节点继续（`tree.go:183-188`）；否则遍历 `n.indices` 找匹配字节，命中则 `incrementChildPrio` 后下沉（`tree.go:191-198`）。
4. 未命中且 `c` 不是 `:`/`*`：追加 `indices` 字符、`addChild` 新节点（`tree.go:201-209`）。
5. 若 `n.wildChild` 为真：下沉到末尾 wildcard 子节点，校验是否同前缀可续走；不满足则 **panic**（wildcard 冲突，如 `:name` 与 `:names`、catchAll 与已有段）（`tree.go:210-235`）。
6. 落到 `insertChild`：`findWildcard` 找 `:param/*`，按 param（建 param 节点+续 `/` 子节点）或 catchAll（仅允许末尾，建空 path 节点+变量节点）建子树；无 wildcard 则直接挂 handler（`tree.go:288-396`）。

**链路 B：匹配一条请求路径（getValue 主路径）**

1. 从根节点 `n` 开始，要求 `path` 以 `n.path` 为前缀（`tree.go:424-425`）。
2. 余下首字节 `idxc`：遍历 `n.indices` 静态匹配；若 `n.wildChild` 为真，**先把当前节点快照压入 `skippedNodes` 栈**再下沉 static 子节点（`tree.go:430-452`）。
3. 静态匹配失败且无 wildcard：若余下不是 `/`，从 `skippedNodes` 弹栈回滚（`strings.HasSuffix` 校验），恢复 path/node/paramsCount 后 `continue walk`（`tree.go:456-473`）；否则给出 tsr 建议返回。
4. 有 wildcard：下沉末尾子节点，`globalParamsCount++`，按 `nType` 分 param（截到 `/` 存参数，`unescape` 时 `url.QueryUnescape`）或 catchAll（存整段剩余）（`tree.go:483-579`）。
5. 走到 `path == prefix`：若节点有 `handlers` 命中返回；否则查 `/` 子节点给 tsr 建议（`tree.go:587-637`）。

**链路 C：大小写不敏感查找（RedirectFixedPath）**

- `findCaseInsensitivePath` 调 `findCaseInsensitivePathRec`：用 `strings.EqualFold` 匹配前缀，对每个 rune 用 `[4]byte` 缓存做小写/大写双分支递归（`tree.go:707-810`）；`wildChild` 为真时**优先静态子节点**再回退 wildcard（`tree.go:821-886`），保证 `/PREFIX/XXX` 命中静态 `/prefix/xxx` 而非 `:id`。

## 4. 配置项

| 配置项 / 行为 | 默认 / 说明 | 位置 |
|------|------|------|
| `unescape` | 由 Engine `UnescapePathValues` 决定；为真时 param 值 `url.QueryUnescape` | `tree.go:513`、`tree.go:566` |
| `RemoveExtraSlash` | Engine 开启时 `rPath = cleanPath`（见 path-utils）后再匹配 | `gin.go:703` |
| `RedirectFixedPath` | Engine 开启时未命中走 `findCaseInsensitivePath` 大小写/斜杠修复重定向 | `gin.go:731`、`tree.go:671` |
| `RedirectTrailingSlash` | 依赖 `getValue` 返回的 `tsr` 标志 | `tree.go:478`、`tree.go:616` |
| `stackBufSize=128` | 大小写查找的栈缓存大小 | `tree.go:8`（path.go 同名常量） |

## 5. 错误与重试语义

- **wildcard 冲突 panic（注册期）**：插入期检测到 `:param` 与已有 wildcard 冲突、catchAll 与已有段冲突、catchAll 不在末尾、转义非法——全部 panic，是开发期编程错误（`tree.go:224-234`、`tree.go:297-305`、`tree.go:344-364`）。
- **重复注册 panic**：同一路径已有 handler 再注册时 panic（`tree.go:242-244`）。
- **无效节点类型 panic**：`getValue`/`findCaseInsensitivePathRec` 的 `default` 分支 `panic("invalid node type")`（`tree.go:582`、`tree.go:934`）。
- **匹配失败 = tsr 标志**：未命中不返回 error，而是 `nodeValue.tsr=true` 建议尾斜杠重定向，由 Engine 决定是否执行；这是"无重试、靠回滚栈兜底"的设计。
- **skippedNode 回滚**：静态子节点走错路时，靠压栈快照恢复，替代传统回溯（`tree.go:460-472`）。

## 6. 并发细节

- **非并发安全注册**：`addRoute` 注释明确"Not concurrency-safe!"（`tree.go:134`），插入期直接改 children/indices/priority，约定所有注册在 `Run` 前单 goroutine 完成。
- **运行期只读**：请求期 `getValue` 只读遍历，不改树结构；多请求 goroutine 并发安全。
- **无锁零分配**：`getValue` 复用 `c.params`/`c.skippedNodes`（由 Engine `allocateContext` 预分配容量，见 engine-lifecycle），不每次分配；`skippedNodes` 栈在单次匹配内复用。
- **priority 热点前置**：`incrementChildPrio` 把高频访问子节点向前移并同步重排 `indices` 字符序（`tree.go:111-131`），使热路径落在 indices 前端——性能优化而非并发原语。
- **context.Context**：本叶子不涉及 context 传播；匹配是纯字符串遍历。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `tree.go` 全文：基数树的插入与匹配。
- `internal/bytesconv`（`BytesToString` 零拷贝，见 internal 边界）。

**Out-of-Scope（不在本仓库源码内）**
- 算法源自 julienschmidt/httprouter（BSD-3，上游仓库）。
- Engine 的 method 选择与 404/405 派发（core-engine）。
- URL 解析（Go 标准库 `net/url`）。

**不做什么**：不选 method 树（Engine 线性遍历 methodTrees）、不执行 handler 链（Context.Next）、不做重定向（Engine 的 redirectRequest）。

## 8. 与相邻子系统交互

- **router-group → 树（注册期）**：`engine.addRoute` 调 `root.addRoute` 插入路由。
- **Engine → 树（运行期）**：`handleHTTPRequest` 调 `root.getValue(rPath, c.params, c.skippedNodes, unescape)`。
- **树 → Context**：`getValue` 把 param 写入 `*c.params`，命中后 Engine 赋 `c.handlers`。
- **Engine → 树（修复重定向）**：`redirectFixedPath` 调 `root.findCaseInsensitivePath`。
- **path-utils → 树**：`cleanPath` 在 `RemoveExtraSlash` 时预处理 rPath。

## 9. 语言专项适配口径

- **非 K8s 控制器**：纯数据结构 + 算法，无 Reconcile/informer。Go 专项重点在"注册期写、运行期只读"的并发切分与零分配复用。
- **internal 边界**：仅引用 `internal/bytesconv` 做 `[]byte→string` 零拷贝（unicode 字符边界正确处理，注释见 #65），不跨仓库暴露。
- **单二进制库形态**：纯内存数据结构，无独立入口。
- **性能模型**：priority 热点重排 + skippedNode 压栈回滚替代回溯 + 复用 Params/skippedNodes 容量，是 gin "高性能零分配路由树"的核心实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 路由树结构与插入/匹配架构图 | `radix-tree-architecture.html` | architecture | standard（降档原因：树状 fan-out 布局的垂直/对角边触发 fromSide/toSide 方向校验，改横向链式后仍 standard，render 退出码 0） |
| getValue 匹配主路径数据流图 | `radix-tree-dataflow.html` | dataflow | standard（降档原因：回滚支路需放到第二行并加宽 viewBox，showcase 严格校验下回退 standard，render 退出码 0） |

- JSON IR 源文件位于 `json/radix-tree-architecture.json`、`json/radix-tree-dataflow.json`。
- dataflow 只画"静态匹配 → wildcard 取值 → 回滚栈兜底 → 命中/tsr"一条主路径；wildcard 冲突 panic 分支在第 3 节文字化展开。
