# 基数树路由匹配（tree）

> 本文是 `routing` 域下的叶子子系统文档。域级总览见 `../routing.md`（输出根 `docs_archify/architecture/`）。
> 本文只展开「按 HTTP 方法分棵的基数树如何插入路由、如何在请求期零分配地匹配路径并提取参数」，
> 不重复展开路由组如何组织注册（见 `../routergroup/routergroup.md`）与路径规范化（见 `../path/path.md`）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。本文件血缘来自
> julienschmidt/httprouter（BSD 许可，见 `tree.go:1-3` 头注）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 按方法分棵 | `methodTree{method, root *node}`，`Engine.trees` 是 `methodTrees` 切片，每个 HTTP 方法一棵独立基数树 | `tree.go:45`、`tree.go:50` |
| 方法树查找 | `methodTrees.get(method)` 线性扫描取树根，未注册过的方法返回 nil | `tree.go:52` |
| 基数树节点 | `node` 持有 `path` 片段、`indices` 子节点首字符串、`children`、`handlers`、`fullPath`、`priority` | `tree.go:99` |
| 节点类型 | `static / root / param(:param) / catchAll(*wildcard)` 四种 `nodeType` | `tree.go:90`、`tree.go:92-97` |
| 路由插入 | `addRoute` 最长公共前缀拆分边、按需新建静态/通配子节点，冲突即 panic | `tree.go:135` |
| 子节点维护 | `addChild` 保证通配子节点恒排在 `children` 末尾；`incrementChildPrio` 按访问频率把高频子节点前移 | `tree.go:71`、`tree.go:111` |
| 通配符识别 | `findWildcard` 定位 `:param`/`*catchAll`，支持 `\:` 转义，校验每段仅一个通配且必须命名 | `tree.go:253` |
| 叶子插入 | `insertChild` 递归把通配前缀拆成 param 节点 + 静态子节点 + catchAll 节点对 | `tree.go:288` |
| 请求匹配 | `getValue` 沿树走查：前缀比对 → indices 静态子节点 → 通配子节点，输出 `nodeValue` | `tree.go:418` |
| 参数提取 | 匹配中把 `:param` 段值、`*catchAll` 剩余路径写入 `Params`，支持 URL 反解码 | `tree.go:487-522`、`tree.go:549-579` |
| 尾斜杠建议 | 未命中时通过 `value.tsr` 标记是否存在「加/去尾斜杠」可命中的路由，供引擎重定向 | `tree.go:478`、`tree.go:533`、`tree.go:616-644` |
| 路径修正回退 | `findCaseInsensitivePath(Rec)` 在大小写/多余斜杠场景递归找可修正路径（RedirectFixedPath） | `tree.go:671`、`tree.go:703` |
| 参数计数 | `countParams`/`countSections` 统计路径参数与段数，供引擎预分配 Context 池容量 | `tree.go:80`、`tree.go:86` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Param` / `Params` | `tree.go:17` / `tree.go:25` | URL 路径参数键值对；`Params.Get`/`ByName` 按名取值 |
| `methodTree` / `methodTrees` | `tree.go:45` / `tree.go:50` | 方法→树根的映射切片；`get(method)` 线性查找 |
| `node` | `tree.go:99` | 基数树节点；`path` 为该节点片段，`indices` 是子节点首字符串（与 `children` 平行），`wildChild` 标记末尾是否为通配子节点 |
| `nodeType` | `tree.go:90` | `static/root/param/catchAll` 枚举，决定 `getValue` 分支 |
| `nodeValue` | `tree.go:400` | `getValue` 返回值：`handlers`、`params`、`tsr`、`fullPath` |
| `skippedNode` | `tree.go:407` | 走静态子节点前暂存的「可回退节点快照」，用于通配冲突时回滚 |
| `addRoute` | `tree.go:135` | 注册期插入主流程（split edge + 通配冲突检测） |
| `getValue` | `tree.go:418` | 请求期匹配主流程（零分配热路径） |

## 3. 关键调用链

**链一：注册期插入一条带参数的路由 `/user/:name`**
1. `routergroup` 调 `engine.addRoute`（`gin.go:364`）后 `root.addRoute(path, handlers)`（`tree.go:135`）。
2. `addRoute` 先 `longestCommonPrefix(path, n.path)`（`tree.go:153`）求最长公共前缀；若公共前缀短于当前节点 `path`，在 `tree.go:156` 处**拆分边**——把当前节点已注册的子树下移为新子节点，当前节点退化为纯前缀节点。
3. 剩余路径首字节若在 `n.indices` 中，`incrementChildPrio`（`tree.go:194`）高频前移后下钻；否则在 `tree.go:237` 调 `insertChild`。
4. `insertChild`（`tree.go:288`）经 `findWildcard`（`tree.go:291`）发现 `:name`，在 `tree.go:314-321` 建 `nType=param` 节点并 `wildChild=true`，其后再挂一个以 `/` 开头的静态子节点。
5. 若与既有通配冲突（如 `/user/:id` 后再插 `/user/:name`），在 `tree.go:230` **panic**；若同一绝对路径重复注册，在 `tree.go:243` **panic**。

**链二：请求期匹配 `GET /user/gin`**
1. `Engine.handleHTTPRequest`（`gin.go:690`）取该方法树根，调 `root.getValue(rPath, c.params, c.skippedNodes, unescape)`（`gin.go:715` → `tree.go:418`）。
2. `getValue` 比对 `n.path` 前缀（`tree.go:425`），命中后取 `path[0]` 在 `n.indices` 中找静态子节点下钻（`tree.go:430-453`）；走静态分支前若 `n.wildChild`，先把当前节点快照压入 `skippedNodes`（`tree.go:433-448`）以备回退。
3. 静态子节点无命中且存在通配子节点时，在 `tree.go:483` 落到末尾通配子节点，`switch n.nType`：`param` 分支（`tree.go:487`）扫描到下一个 `/` 切段，把 `path[:end]` 写入 `Params`（`tree.go:518`）；`catchAll` 分支（`tree.go:549`）把剩余整段写入 `Params`。
4. 命中 `handlers` 即填 `value.handlers/fullPath` 返回（`tree.go:537-539`、`tree.go:608-610`）；未命中则置 `value.tsr`（`tree.go:478`、`tree.go:642`），由引擎决定是否做尾斜杠重定向。

**链三：通配回退（skippedNode）**
1. 当静态分支走错（如 `/user/gin` 实际应落到 `/user/:name`），`getValue` 在 `tree.go:460-472` 弹栈 `skippedNodes`，恢复之前的节点快照与 `paramsCount`，回滚 Params 长度后继续 walk。这解决了「静态路径与通配路径共享前缀时的匹配优先级」。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `Engine.RedirectTrailingSlash` | 默认 `true`；`getValue` 产出 `tsr=true` 时引擎做 301/307 尾斜杠重定向 | `gin.go:211`、`gin.go:727` |
| `Engine.RedirectFixedPath` | 默认 `false`；开启后 `redirectFixedPath` 调 `findCaseInsensitivePath` 做大小写/斜杠修正重定向 | `gin.go:212`、`gin.go:731` |
| `Engine.HandleMethodNotAllowed` | 默认 `false`；开启后对其他方法树再 `getValue` 一次以收集 `Allow` 头（`gin.go:746`） | `gin.go:213` |
| `Engine.UseRawPath` / `UseEscapedPath` / `UnescapePathValues` | 决定 `getValue` 收到的 `rPath` 是 RawPath/EscapedPath，以及 `param` 值是否 `url.QueryUnescape` | `gin.go:695-701`、`tree.go:513-517` |
| `engine.maxParams` / `maxSections` | 注册期累加的路径参数/段数最大值，用于 Context 池预分配 Params 容量 | `gin.go:379-385`、`tree.go:80` |
| 通配符语法约束 | 每段仅一个通配、通配必须命名、`*catchAll` 必须在路径末尾且前导 `/`、`\:` 转义 | `tree.go:298`、`tree.go:304`、`tree.go:345`、`tree.go:363` |

## 5. 错误与重试语义

- **注册期冲突一律 panic**：通配冲突（`tree.go:230`）、重复注册（`tree.go:243`）、非法通配语法（`tree.go:262`、`tree.go:298`、`tree.go:304`、`tree.go:345`、`tree.go:363`）。与 routergroup 一致，启动期 fail-fast。
- **匹配期无错误返回**：`getValue` 是纯函数，不返回 error；未命中用 `nodeValue.handlers==nil` + `tsr` 标志表达，由 `handleHTTPRequest` 决定重定向/405/404。
- **反解码容错**：`url.QueryUnescape` 失败时（`tree.go:514`、`tree.go:567`）保留原始字节，不报错、不中断匹配。
- **无重试/退避**：纯内存查找，无外部依赖，无重试概念。

## 6. 并发细节

- **注册非并发安全**：`tree.go:134` 明确 `Not concurrency-safe!`。`addRoute`/`insertChild`/`incrementChildPrio` 修改 `children`/`indices`/`priority`，必须在监听前单线程完成。
- **运行期只读**：`getValue` 遍历不可变树，多 goroutine 并发安全。`skippedNodes` 与 `params` 是 per-request 栈/池对象（挂在 `Context` 上），不跨请求共享。
- **零分配目标**：`getValue` 尽量复用传入的 `*Params` 与 `*skippedNode` 切片容量（`tree.go:500-504` 仅在容量不足时扩容），`fullPath`/`path` 用字符串切片共享底层数组——这是 gin「零分配路由」的核心。
- 无 goroutine、无 channel；`incrementChildPrio` 在注册期一次性完成子节点排序，运行期无锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `tree.go`：基数树节点、插入、匹配、通配处理、tsr、大小写修正查找。
- `gin.go` 中 `Engine.trees` 字段（`gin.go:184`）、`addRoute` 的树注册（`gin.go:371-377`）、`handleHTTPRequest` 的匹配调用（`gin.go:690-760`）。

**Out-of-Scope（不在本仓库源码内 / 相邻叶子覆盖）**
- 路由组与中间件链拼装 → 见 `../routergroup/routergroup.md`。
- 请求前的 `cleanPath` 规范化 → 见 `../path/path.md`。
- 匹配到 `handlers` 后 `Context.Next()` 的中间件执行 → context 域。
- `net/url.QueryUnescape` 为标准库（外部，不在本仓库源码内）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：注册期 `Engine.addRoute`（`gin.go:364`）写入；请求期 `Engine.handleHTTPRequest`（`gin.go:690`）查询。
- 本叶子 → 下游：`getValue` 把 `handlers HandlersChain`、`Params`、`fullPath` 回填到 `Context`（`gin.go:716-724`），交 context 域执行中间件链。
- 配置交互：`RemoveExtraSlash` 开启时，`handleHTTPRequest` 先调 `path` 叶子的 `cleanPath` 再查树；`RedirectFixedPath` 开启时调本叶子的 `findCaseInsensitivePath`。

## 9. 语言专项适配口径

- **并发模型**：非 k8s 控制器模式；是「启动期单线程构建不可变基数树 + 运行期多 goroutine 只读查找」。`context.Context` 不贯穿树本身（树查找不感知超时），仅在 `Context` 层承载请求取消。
- **数据结构即状态机**：`nodeType` 的 `param/catchAll` 分支是 `getValue` 内的隐式状态分发；`skippedNodes` 栈实现「静态优先、通配兜底」的回退，可视为匹配过程的小型内部状态机（未单独画 lifecycle，因它是单次 walk 内的局部回退，非长生命周期实体）。
- **多二进制**：无 `cmd/`，纯库；本叶子是 `Engine` 内部数据结构，不独立部署。
- **internal 边界**：仅用 `internal/bytesconv.BytesToString`（`tree.go:13`、`tree.go:170`、`tree.go:203`）做零拷贝字节串转换，未越界。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 基数树路由结构架构图 | `tree-architecture.html` | architecture | showcase |
| getValue 路由匹配数据流 | `tree-dataflow.html` | dataflow | **standard** |

JSON IR 位于 `json/` 目录。

**standard 档披露**：`tree-dataflow.html` 两轮 showcase 校验后仍落 standard。失败检查名：`composition/micro-segment`（flows[4] `n4→n6` 出现 7px 内部微段，低于 8px 最小分段）；首轮另有 `composition/label-route-clearance`（「通配取值」与「命中 handlers」标签间距不足）。采取的修复动作：① 加宽 `meta.viewBox` 至 1200；② 缩短节点标签（如「indices 命中静态子节点」→「indices 静态命中」）；③ 删除引发跨节点穿越的 `n2→n6` bottom-channel 边，改由 `n4→n6` 主路径表达参数入 Params；④ `n1→n3` 改 `vertical-channel`。仍残留一条亚像素微段边，按「≥2 轮不过落 standard」处理。
本叶子未生成 sequence / workflow / lifecycle 图：注册与匹配的跨对象调用时序已由 routergroup 叶子的 sequence 覆盖；无多角色泳道流程；`node` 是静态数据结构而非长生命周期实体，其匹配内部分支用 dataflow 管道表达更贴切，按资源节省原则省略。
