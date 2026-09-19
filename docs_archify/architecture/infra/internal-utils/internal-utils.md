# internal 零拷贝与文件系统辅助（internal-utils）

> 本文是 `infra` 域下的叶子子系统文档。域级总览见 `../infra.md`，
> 本文只展开 `internal/bytesconv` 零拷贝转换、`internal/fs` 适配与根包 `fs.go` 的静态文件辅助，
> 不重复 RouterGroup 静态文件注册流程（见 routergroup 叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `internal/bytesconv/bytesconv.go`（22 行）、`internal/fs/fs.go`（22 行）、`fs.go`（51 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `bytesconv.StringToBytes` | 无分配把 string 转 []byte | `internal/bytesconv/bytesconv.go:13` |
| `bytesconv.BytesToString` | 无分配把 []byte 转 string | `internal/bytesconv/bytesconv.go:19` |
| `fs.FileSystem` | 把 `http.FileSystem` 适配成 `fs.FS` | `internal/fs/fs.go:9` |
| `OnlyFilesFS` | 关闭目录列表的 http.FileSystem 包装 | `fs.go:13` |
| `Dir(root, listDirectory)` | 构造静态文件服务用的 FileSystem | `fs.go:42` |

对外暴露点：`bytesconv` 被 auth/recovery/logger 等热路径内部使用；`Dir` 被 `RouterGroup.Static` 调用。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `StringToBytes` | `internal/bytesconv/bytesconv.go:13` | `unsafe.Slice(unsafe.StringData(s), len(s))` |
| `BytesToString` | `internal/bytesconv/bytesconv.go:19` | `unsafe.String(unsafe.SliceData(b), len(b))` |
| `fs.FileSystem` | `internal/fs/fs.go:9` | 内嵌 `http.FileSystem`，实现 `fs.FS` |
| `OnlyFilesFS` | `fs.go:13` | 内嵌 `http.FileSystem` |
| `neutralizedReaddirFile` | `fs.go:28` | 包装 `http.File`，`Readdir` 恒返回 nil |

设计要点：`internal/` 目录在 Go 构建约束下禁止外部项目 import，这些工具仅 Gin 自身可用。

## 3. 关键调用链

**主路径：零拷贝转换（`StringToBytes`，`internal/bytesconv/bytesconv.go:13`）**

1. 调用方传入 string `s`。
2. `unsafe.StringData(s)` 取字符串底层数据指针（`bytesconv.go:14`）。
3. `unsafe.Slice(ptr, len(s))` 把该指针重解释为 `[]byte`，**不分配新内存**（`bytesconv.go:14`）。
4. 反向 `BytesToString` 用 `unsafe.String(unsafe.SliceData(b), len(b))`（`bytesconv.go:20`）——只读转换，调用方不得修改返回的 []byte。

**主路径：http.FileSystem → fs.FS 适配（`fs.FileSystem.Open`，`internal/fs/fs.go:14`）**

1. `o.FileSystem.Open(name)` 委托上游打开（`fs.go:15`）。
2. 出错返回 nil, err（`fs.go:16-18`）。
3. 成功后 `fs.File(f)` 把 `http.File` 接口值直接当 `fs.File` 返回（`fs.go:20`）——两者方法集兼容，仅类型转换。

**主路径：构造无目录列表的静态 FileSystem（`Dir`，`fs.go:42`）**

1. `fs := http.Dir(root)`（`fs.go:43`）。
2. `listDirectory == true` 直接返回 `http.Dir`（`fs.go:45-47`）。
3. 否则包成 `&OnlyFilesFS{FileSystem: fs}`（`fs.go:49`）。
4. `OnlyFilesFS.Open` 打开文件后包成 `neutralizedReaddirFile`（`fs.go:18-24`），其 `Readdir` 恒返回 `nil, nil`（`fs.go:33-35`），关闭 `http.FileServer` 的目录列表。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `Dir(root, listDirectory)` | `listDirectory=false` 时关闭目录列表 | `fs.go:42-49` |
| 零拷贝安全性 | 调用方必须保证返回的 []byte 不被修改且不超过原 string 生命周期 | `bytesconv.go:11-12` |

## 5. 错误与重试语义

- `fs.FileSystem.Open` / `OnlyFilesFS.Open` 把上游 `Open` 的错误原样透传（`internal/fs/fs.go:16`、`fs.go:19-21`）。
- `neutralizedReaddirFile.Readdir` 不报错，恒返回 `nil, nil` 以禁用列表。
- 零拷贝转换本身不返回错误，但 `unsafe` 越界使用会产生未定义行为——Gin 仅在"只读、短生命周期"场景使用。
- 无重试。

## 6. 并发细节

- 不创建 goroutine。
- 零拷贝函数无状态、无锁，纯函数。
- `unsafe.Slice/unsafe.String` 依赖 Go 内存模型的只读引用；Gin 在日志/认证热路径用它避免分配。
- `OnlyFilesFS`/`fs.FileSystem` 是无状态值类型，可并发调用。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `internal/bytesconv`、`internal/fs`、根包 `fs.go`。

**Out-of-Scope（不在本仓库源码内）**
- `unsafe`、`io/fs`、`net/http`、`os` 标准库：指针重解释、FS 接口、文件系统，外部依赖。
- 静态文件服务的路由注册在 routergroup 叶子；`http.FileServer` 是标准库实现。
- 本叶子不实现模板加载（engine-lifecycle 叶子）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：auth（`auth.go:13`、`auth.go:37`）、recovery（`recovery.go:23`、`recovery.go:100`）调用 `bytesconv`。
- 上游 → 本叶子：`RouterGroup.Static`（routergroup 叶子）调用 `Dir` 构造 FileSystem。
- 本叶子 → 下游：`internal/fs` 把 `http.FileSystem` 适配给 `template.ParseFS` 等需要 `fs.FS` 的 API。

## 9. 语言专项适配口径

- **Go internal 边界**：`internal/` 目录是 Go 编译器级访问控制，确保这些 unsafe 工具只被 Gin 自身使用，不泄漏给外部依赖者——这是 Gin 控制零拷贝安全风险的关键边界。
- **unsafe 现代用法**：用 `unsafe.StringData/unsafe.SliceData` + `unsafe.Slice/unsafe.String`（Go 1.20/1.20+），替代早期 `reflect.StringHeader` 技巧，更安全（见 `bytesconv.go:11` 注释引用的 go issue）。
- **接口适配**：`internal/fs.FileSystem` 仅做 `http.FileSystem`→`fs.FS` 的接口类型适配，无逻辑。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| internal 工具架构图 | `internal-utils-architecture.html` | architecture | showcase |
| JSON IR | `json/internal-utils-architecture.json` | — | — |

本叶子不补时序图：转换/适配均为无状态纯函数，架构图已表达 internal 边界与依赖方向。
