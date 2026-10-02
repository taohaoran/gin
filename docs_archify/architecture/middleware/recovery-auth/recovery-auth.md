# 崩溃恢复与 BasicAuth 认证中间件（recovery-auth）

> 本文是 `middleware` 域下的叶子子系统文档。域级总览见 [`../middleware.md`](../middleware.md)。
> 本文合并两个源码文件：崩溃恢复 `recovery.go` 与 Basic 认证 `auth.go`——二者都是"包在 `c.Next()`
> 外围/链中"的防御性中间件。链执行协议见
> [`../middleware-core/middleware-core.md`](../middleware-core/middleware-core.md)。
>
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 崩溃恢复中间件 | `Recovery()` 捕获下游 panic，写 500，防止整个 server 崩 | `recovery.go:35` |
| 自定义恢复动作 | `CustomRecovery(handle)` 注入 `RecoveryFunc` 替代默认 500 | `recovery.go:40` |
| 指定输出 writer | `RecoveryWithWriter(out, ...)` 指定日志输出，可接错误 writer | `recovery.go:45` |
| 可插拔恢复处理器 | `CustomRecoveryWithWriter(out, handle)`：defer recover + 日志 + 调用 handle | `recovery.go:53` |
| 断连特判 | 识别 EPIPE/ECONNRESET/`http.ErrAbortHandler`，不打完整堆栈、只 Abort | `recovery.go:63-69`、`recovery.go:81` |
| 脱敏请求转储 | `secureRequestDump` 把 Authorization 头打码为 `Authorization: *` | `recovery.go:98` |
| 堆栈打印 | `stack(skip)` 逐帧 `runtime.Caller` + `readNthLine` 读源码行 + `function` 美化函数名 | `recovery.go:119`、`recovery.go:151`、`recovery.go:177` |
| 默认恢复处理 | `defaultHandleRecovery`：`c.Error(e)` + `AbortWithStatus(500)` | `recovery.go:109` |
| BasicAuth 中间件 | `BasicAuth(accounts map[user]pass)` 校验 `Authorization` 头 | `auth.go:72` |
| 指定 Realm | `BasicAuthForRealm(accounts, realm)`，空 realm 用 "Authorization Required" | `auth.go:48` |
| 代理认证 | `BasicAuthForProxy` 校验 `Proxy-Authorization`，失败回 407 | `auth.go:98` |
| 常量时间比较 | `authPairs.searchCredential` 用 `subtle.ConstantTimeCompare` 防爆破 | `auth.go:32`、`auth.go:37` |
| 凭证预编译 | `processAccounts` 把 user:pass 预编码成 `Basic base64` 串切片 | `auth.go:76`、`auth.go:91` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `RecoveryFunc` | `recovery.go:32` | `func(c *Context, err any)`，自定义 panic 处理器签名 |
| `CustomRecoveryWithWriter` 返回闭包 | `recovery.go:58` | 真正的中间件：`defer recover()` 包 `c.Next()` |
| `Accounts` | `auth.go:23` | `map[string]string`，用户名→密码 |
| `authPair` / `authPairs` | `auth.go:25` / `auth.go:30` | 预编译凭证：`value="Basic base64(...)"` → `user`；切片线性扫描做常量时间比较 |
| `AuthUserKey` / `AuthProxyUserKey` | `auth.go:17` / `auth.go:20` | 认证成功后写入 `c.Keys` 的键名（`"user"`/`"proxy_user"`） |

## 3. 关键调用链

### 3.1 Recovery：panic 如何被转成 500

1. `Recovery()`（`recovery.go:35`）→ `RecoveryWithWriter(DefaultErrorWriter)`（`recovery.go:45`）→ 无自定义 handle 时走 `defaultHandleRecovery`（`recovery.go:49`）。
2. 中间件闭包进入后立刻 `defer func(){ if rec := recover(); rec != nil { ... } }()`（`recovery.go:59-60`），然后才 `c.Next()`（`recovery.go:90`）执行下游链。
3. 下游任一处理器 panic 时，`recover()` 捕获：先断言 `rec.(error)` 并判断是否断连（`errors.Is(err, syscall.EPIPE/ECONNRESET/http.ErrAbortHandler)`，`recovery.go:66-68`）。
4. 日志：断连只打请求转储；非断连且 `IsDebugging()` 时打请求转储 + panic 值 + 堆栈（`recovery.go:74`）；否则只打 panic 值 + 堆栈（`recovery.go:77`）。
5. 处置：断连时 `c.Error(err)` + `c.Abort()`（`recovery.go:83-84`，连接已死不再写状态）；否则调 `handle(c, rec)`——默认 `defaultHandleRecovery`（`recovery.go:109`）把 `rec` 包成 error 后 `c.Error(e)` + `c.AbortWithStatus(http.StatusInternalServerError)`（`recovery.go:115`）。

### 3.2 堆栈转储的生成

1. `stack(stackSkip)`（`recovery.go:119`，`stackSkip=3` `recovery.go:28`）从 `runtime.Caller(i)` 逐帧取 `pc/file/line`。
2. 换文件时 `readNthLine(file, line-1)`（`recovery.go:151`）用 `bufio.Scanner` 读源码对应行，缓存 `lastFile` 避免重复开文件；`function(pc)`（`recovery.go:177`）用 `runtime.FuncForPC` 去掉包路径前缀并把 `·` 替换为 `.`，得到短函数名。

### 3.3 BasicAuth：请求如何被鉴权放行/拒绝

1. `BasicAuth(accounts)`（`auth.go:72`）→ `BasicAuthForRealm(accounts, "")`（`auth.go:48`）：空 realm 填默认值，拼成 `Basic realm="..."` 后调 `processAccounts`（`auth.go:76`）把每个 `user:pass` 预编码成 `Basic base64(user:pass)` 存入 `authPairs`。
2. 请求进入闭包：`pairs.searchCredential(c.requestHeader("Authorization"))`（`auth.go:56`）取请求头。
3. `searchCredential`（`auth.go:32`）遍历预编译切片，用 `subtle.ConstantTimeCompare`（`auth.go:37`，经 `bytesconv.StringToBytes` 零拷贝转字节）与请求头比较——常量时间比较防时序侧信道。
4. 未命中：写 `WWW-Authenticate` 头 + `c.AbortWithStatus(401)`（`auth.go:59-60`），链在此截断。
5. 命中：`c.Set(AuthUserKey, user)`（`auth.go:66`）把用户名存入 `c.Keys`，**不调用 `c.Next()` 之外的额外动作，自然放行后续链**。
6. `BasicAuthForProxy`（`auth.go:98`）同构，只是读 `Proxy-Authorization` 头、失败回 407、键名用 `AuthProxyUserKey`。

## 4. 配置项

| 配置 / 选项 | 默认 / 行为 | 位置 |
|---|---|---|
| Recovery 输出 writer | `Recovery()` 用 `DefaultErrorWriter`；可经 `RecoveryWithWriter` 替换 | `recovery.go:36`、`recovery.go:45` |
| 自定义恢复函数 | `CustomRecovery`/`RecoveryWithWriter` 可变参数传入；缺省 `defaultHandleRecovery` | `recovery.go:40`、`recovery.go:49` |
| 调试模式堆栈详情 | `IsDebugging()` 为 true 时日志额外打印脱敏请求转储 | `recovery.go:73` |
| BasicAuth realm | 空串时 HTTP 用 "Authorization Required"，代理用 "Proxy Authorization Required" | `auth.go:50`、`auth.go:100` |
| 空账号列表 | `processAccounts` 用 `assert1(length>0, ...)` 启动期 panic | `auth.go:78` |
| 堆栈跳过帧数 | 常量 `stackSkip=3` | `recovery.go:28` |

## 5. 错误与重试语义

- **Recovery 是 panic 的唯一防线**：它把 panic 转化为受控的 500 响应与一条日志；**panic 不会被重试**。若业务 panic 后需要重试，应由上游（反向代理/客户端）负责，Recovery 自身不做重试。
- **断连是预期内失败**：EPIPE/ECONNRESET 不打完整堆栈、不写状态码（连接已死），只 `c.Abort()` 并把错误推进 `c.Errors`（`recovery.go:83`）。
- **BasicAuth 失败是正常 401**：不是 error，不进 `c.Errors`，直接 `AbortWithStatus(401)` 截断链。
- **`c.Error(nil)` 防护**：Recovery 调 `c.Error(err)` 时 err 必非 nil（recover 出的 any 被 `fmt.Errorf("%v")` 包装，`recovery.go:112`）；而 `Context.Error` 对 nil 会 panic（`context.go:263`）。

## 6. 并发细节

- **Recovery 的 defer/recover 是同步栈机制**：不创建 goroutine，panic 在当前 goroutine 被恢复，`c.Next()` 之后的中间件后置逻辑不再执行（因为 defer 直接 return 出了闭包）。
- **`stack()` 读源码文件**：每次 panic 都同步打开 `.go` 文件读行（`recovery.go:156`），在高频 panic 场景下是 IO 开销；`lastFile` 缓存仅在同一堆栈内去重，不跨请求缓存。
- **BasicAuth 无锁**：`authPairs` 在装配期一次性构建（`auth.go:76`），请求期只读遍历；`c.Set(AuthUserKey, ...)`（`auth.go:66`）走 `Context.mu` 写锁（`context.go:287`）。
- **`subtle.ConstantTimeCompare`**：消除分支时序差异，但仍需遍历全部凭证（线性扫描），凭证量大时需注意复杂度。
- 不持有 `context.Context`；panic 后请求 goroutine 正常结束，无泄漏。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `recovery.go`：Recovery/CustomRecovery/RecoveryWithWriter/CustomRecoveryWithWriter、defaultHandleRecovery、secureRequestDump、stack/readNthLine/function/timeFormat。
- `auth.go`：BasicAuth/BasicAuthForRealm/BasicAuthForProxy、processAccounts/searchCredential/authorizationHeader、Accounts/常量键。

**Out-of-Scope（不在本仓库源码内）**
- `syscall.EPIPE/ECONNRESET`、`crypto/subtle.ConstantTimeCompare`、`encoding/base64`、`net/http/httputil.DumpRequest`、`runtime.Caller/FuncForPC` 均为 Go 标准库，不在本仓库源码内。
- `internal/bytesconv.StringToBytes` 零拷贝转换见 support 域 internal-utils 叶子。
- `c.Error/AbortWithStatus` 的链截断语义见 middleware-core 叶子；`c.Errors` 聚合见 support/errors。
- 认证后端（数据库/LDAP/JWT）不在本仓库——gin 只提供内存 BasicAuth。

## 8. 与相邻子系统交互

- **上游**：链协议把 Recovery 放在链首（`Default()` `gin.go:239` 注册 Logger+Recovery），用 defer 包住 `c.Next()`；BasicAuth 通常由用户挂在需要保护的路由组上（`r.Group("/admin", BasicAuth(accounts))`）。
- **下游**：Recovery 兜底保护 Logger 之后的所有业务处理器；BasicAuth 失败时 `AbortWithStatus` 截断下游业务处理器。
- **错误交互**：Recovery 把 panic 推进 `c.Errors`（support/errors），Logger 在后置阶段读取 `ErrorTypePrivate` 错误打印（`logger.go:302`）——三者通过 `c.Errors` 解耦协作。

## 9. 语言专项适配口径（Go）

- **panic/recover 惯用法**：Recovery 是 Go Web 框架的标准防御模式——`defer` + `recover()` 把 goroutine panic 转化为 HTTP 500，避免进程崩溃。这是 Go 独有的错误处理范式（与 Java 的 try-catch 等价），不涉及 K8s 控制器模式。
- **internal 边界**：两文件都 import `internal/bytesconv`（`recovery.go:23`、`auth.go:13`）做零拷贝 string↔[]byte——这是 `internal/` 包被本仓库内正常使用的示例，未被外部越界 import。
- **常量时间比较**：`subtle.ConstantTimeCompare` 是 Go 安全编程惯例，用于密码比较防时序攻击。
- **多二进制**：纯库中间件，无独立二进制。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Recovery 与 BasicAuth 组件图 | [recovery-auth-architecture.html](recovery-auth-architecture.html) | architecture | showcase |
| panic 恢复链路时序图 | [recovery-auth-sequence.html](recovery-auth-sequence.html) | sequence | showcase |
| panic 处置状态流转图 | [recovery-auth-lifecycle.html](recovery-auth-lifecycle.html) | lifecycle | showcase |

JSON IR 源文件位于 `json/` 目录。

**省略说明**：本叶子未生成 workflow/dataflow 图——Recovery/BasicAuth 是同步中间件（已由 sequence 表达），无多角色泳道流程、无数据 ETL 管道；lifecycle 图表达"一次 panic 的处置分支"这一单一实体状态流转。
