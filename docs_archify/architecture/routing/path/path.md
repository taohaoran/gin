# 路径规范化（path）

> 本文是 `routing` 域下的叶子子系统文档。域级总览见 `../routing.md`（输出根 `docs_archify/architecture/`）。
> 本文只展开「请求路径在进入基数树匹配前的规范化算法」，不重复展开基数树匹配（见 `../tree/tree.md`）。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。`cleanPath` 血缘来自
> Go 标准库 `path` 包（见 `path.go:1-4` 头注）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| URL 规范化 | `cleanPath(p)` 输出规范 URL 路径：合并多斜杠、消除 `.`/`..`、补前导 `/`、保留尾斜杠 | `path.go:23` |
| 懒建缓冲 | `bufApp` 仅在原路径确需修改时才分配缓冲，常见情况零分配返回子串 | `path.go:128` |
| 去重复字符 | `removeRepeatedChar(s, '/')` 把连续多个 `/` 压成单个（用于重定向前缀清洗） | `path.go:155` |
| 栈缓冲容量 | `stackBufSize=128`，短路径在栈上预分配缓冲避免堆分配 | `path.go:8` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `cleanPath` | `path.go:23` | 纯函数输入路径串、输出规范路径串；`r`/`w` 双指针原位扫描 |
| `bufApp` | `path.go:128` | 惰性写缓冲辅助：首字节与原串相同时不分配，否则复制已有前缀到缓冲 |
| `removeRepeatedChar` | `path.go:155` | 先线性探测是否存在连续重复字符，无则原样返回；有则用缓冲压缩 |

## 3. 关键调用链

**链一：`cleanPath` 规范化一次请求路径**
1. 入口 `path.go:23`：空串直接返回 `/`（`path.go:25`）；记录 `trailing` 尾斜杠标志（`path.go:54`）。
2. 若 `p[0] != '/'`，补前导 `/` 并把读写指针 `r/w` 从 0/1 开始（`path.go:43-52`）。
3. 主循环 `path.go:61-109` 按字节扫描：
   - `path.go:63` 遇到 `/` → 跳过空段（多斜杠合并）；
   - `path.go:67` 末尾 `.` → 置 `trailing=true`；
   - `path.go:71` 命中 `./` → 跳过 `.` 段；
   - `path.go:75` 命中 `..` → `r+=3` 后在 `path.go:79-92` 把写指针 `w` 回退到上一个 `/`（消除父目录段）；
   - `path.go:94` 普通段 → 经 `bufApp`（`path.go:98`、`path.go:104`）逐字节写入。
4. 末尾 `path.go:112-115` 按需补回尾斜杠；`path.go:120-123` 若全程未改缓冲则返回原串子串（零分配），否则返回 `string(buf[:w])`。

**链二：调用点**
1. `Engine.handleHTTPRequest` 在 `gin.go:703-705` 当 `engine.RemoveExtraSlash=true` 时调 `cleanPath(rPath)`，结果作为 `getValue` 的输入路径。
2. `removeRepeatedChar` 不在 `cleanPath` 内部调用，而在 `redirectTrailingSlash`（`gin.go:786`）清洗 `X-Forwarded-Prefix` 时使用。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|---|---|---|
| `Engine.RemoveExtraSlash` | 默认 `false`；开启后请求路径先经 `cleanPath` 再进基数树（使 `//foo//bar` 可匹配 `/foo/bar`） | `gin.go:219`、`gin.go:703` |
| `stackBufSize` | 常量 128；短路径用栈缓冲，超长才堆分配 | `path.go:8` |

本叶子无独立 flag；`RemoveExtraSlash` 定义在 `gin.go`，是唯一触发开关。

## 5. 错误与重试语义

- 纯字符串处理，无错误返回、无外部 IO、无重试。
- `cleanPath` 对 `..` 的回退有边界保护：`w > 1` 才回退（`path.go:79`），避免把根路径回退到 `/` 以下；开头的 `/..` 按规则归一为 `/`。
- 不 panic、不返回 error；任何输入都产出一个合法路径串。

## 6. 并发细节

- 纯函数，无共享状态、无锁、无 goroutine；`buf` 为每次调用局部变量，天然并发安全。
- 零分配设计：`bufApp`（`path.go:130-136`）在「原路径未被修改」时直接返回原串子串，避免短路径规范化产生堆分配——这是 gin 路由零分配目标的一部分。
- 请求期在 `handleHTTPRequest` 主 goroutine 内同步调用，不跨 goroutine 传递。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `path.go`：`cleanPath`/`bufApp`/`removeRepeatedChar`。

**Out-of-Scope（不在本仓库源码内 / 相邻叶子覆盖）**
- 规范化之后的基数树匹配 → 见 `../tree/tree.md`。
- `path.Join`（`joinPaths` 注册期用）在 `utils.go:136`，不在本文件。
- Go 标准库 `path.Clean` 为外部参考实现（不在本仓库源码内）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：`Engine.handleHTTPRequest`（`gin.go:690`）在 `RemoveExtraSlash` 开启时把 `rPath` 送入 `cleanPath`。
- 本叶子 → 下游：规范化后的路径作为 `node.getValue`（`tree.go:418`）的输入。
- 旁支：`removeRepeatedChar` 供重定向路径前缀清洗使用（`gin.go:786`），与匹配主路径解耦。

## 9. 语言专项适配口径

- **并发模型**：无 goroutine/channel；纯函数零共享可变状态，是最易并发安全的组件。
- **控制器模式差异**：与 k8s Reconciler 无关；是同步字符串处理工具。
- **多二进制**：无 `cmd/`，纯库函数。
- **internal 边界**：本文件不依赖 `internal/`，仅用标准库逻辑；与 `internal/fs`（另一处路径清理）职责分离，不交叉。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 路径规范化组件架构图 | `path-architecture.html` | architecture | showcase |
| cleanPath 规范化数据流 | `path-dataflow.html` | dataflow | **standard** |

JSON IR 位于 `json/` 目录。

**standard 档披露**：`path-dataflow.html` 两轮 showcase 校验后落 standard。失败检查名：`composition/proper-crossing`（`n4→n5` 回退边与 `n2→n6`、`n3→n6` 输出边在通道上交叉）与 `composition/label-route-clearance`（「命中 .」与「命中 ..」两条出发边标签间距 0px）。采取的修复动作：① `n2→n6` 改 `bottom-channel` 绕开中段节点；② `n1→n3`/`n1→n4` 改 `vertical-channel`；③ 加宽 `meta.viewBox` 至 1200。三分支扫描（多斜杠/`.`/`..`）天然在同一列发散，showcase 对交叉与标签间距要求过严，两轮后按协议落 standard。
本叶子未生成 sequence / workflow / lifecycle 图：`cleanPath` 是单输入单输出的纯函数扫描，无多参与者消息时序、无多角色泳道、无实体状态机；其组件组成与扫描数据流已由 architecture + dataflow 表达，按资源节省原则省略。
