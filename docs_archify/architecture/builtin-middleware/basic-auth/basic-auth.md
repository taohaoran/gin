# BasicAuth 基本认证中间件（basic-auth）

> 本文是 `builtin-middleware` 域下的叶子子系统文档。域级总览见 `../builtin-middleware.md`，
> 本文只展开 HTTP Basic 认证中间件，不重复日志/恢复中间件（见各自叶子）。
>
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`，文件 `auth.go`（117 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BasicAuth(accounts)` | 最常用入口，默认 realm "Authorization Required" | `auth.go:72` |
| `BasicAuthForRealm(accounts, realm)` | 指定 realm 的 Basic 认证中间件核心实现 | `auth.go:48` |
| `BasicAuthForProxy(accounts, realm)` | 代理认证，读 `Proxy-Authorization`，失败 407 | `auth.go:98` |
| `Accounts` | `map[string]string` 用户→口令表 | `auth.go:23` |
| 凭证预计算 | `processAccounts` 把 user:pass 预编码为 `Basic xxx` | `auth.go:76` |
| 常量时间比对 | `subtle.ConstantTimeCompare` 防时序攻击 | `auth.go:37` |
| 用户名回写 | 认证成功后 `c.Set(AuthUserKey, user)` | `auth.go:66` |

对外暴露点：`BasicAuth` 通常由用户 `r.Use(BasicAuth(accounts))` 或挂到某个 `RouterGroup` 下。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Accounts` | `auth.go:23` | `map[string]string`，键用户名、值口令 |
| `authPair` | `auth.go:25` | 内部结构：预编码后的 `value` 与对应 `user` |
| `authPairs` | `auth.go:30` | `[]authPair`，持有 `searchCredential` 方法 |
| `searchCredential` | `auth.go:32` | 遍历凭证表，常量时间比对请求头值 |
| `authorizationHeader` | `auth.go:91` | 把 `user:password` 做 base64 编码成 `Basic xxx` |
| `processAccounts` | `auth.go:76` | 启动期校验空表/空用户并预编码 |
| `AuthUserKey` / `AuthProxyUserKey` | `auth.go:17`、`auth.go:20` | `c.Keys` 中存取用户名的键常量 |

扩展点：无回调式扩展；认证后用户名固定写入 `c.Keys[AuthUserKey]`，下游用 `c.MustGet(gin.AuthUserKey)` 读取。

## 3. 关键调用链

**主路径：一次 Basic 认证请求（`BasicAuthForRealm` 返回的闭包）**

1. 中间件被调用，取请求头：`pairs.searchCredential(c.requestHeader("Authorization"))`（`auth.go:56`；`requestHeader` 实现在 `context.go:1100`，即 `Header.Get`）。
2. 未命中（`!found`）：写 `WWW-Authenticate` 响应头并 `c.AbortWithStatus(401)` 中止链（`auth.go:59-60`；`AbortWithStatus` 在 `context.go:223`）。
3. 命中：把用户名写入上下文 `c.Set(AuthUserKey, user)`（`auth.go:66`；`c.Set` 在 `context.go:286`，受 `mu` 锁保护）。
4. 下游 handler 通过 `c.MustGet(gin.AuthUserKey)` 取回用户名（`MustGet` 在 `context.go:306`，键不存在会 panic）。

**启动期凭证预计算（`processAccounts`，`auth.go:76`）**

1. `assert1(length > 0, "Empty list...")`：空表直接 panic（`auth.go:78`；`assert1` 在 `utils.go:86`）。
2. 遍历 map，`assert1(user != "", ...)` 拒绝空用户名（`auth.go:81`）。
3. 对每条调用 `authorizationHeader(user, password)`：`user+":"+password` → base64 → `"Basic "+encoded`（`auth.go:82-93`；base64 输入用 `bytesconv.StringToBytes` 零拷贝，`auth.go:93`）。

**常量时间比对（`searchCredential`，`auth.go:32`）**

- 空值直接返回未命中（`auth.go:33-35`）。
- 逐对 `subtle.ConstantTimeCompare(预编码值字节, 请求头值字节)`，返回 1 即命中（`auth.go:37`）——避免按字符比较早停带来的时序侧信道。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `accounts` | 必传，空表启动期 panic | `auth.go:78` |
| `realm` | 空时 Basic 用 `"Authorization Required"`，Proxy 用 `"Proxy Authorization Required"` | `auth.go:49-51`、`auth.go:99-101` |
| realm 格式 | `"Basic realm=" + strconv.Quote(realm)` | `auth.go:52` |
| 认证头 | 普通认证读 `Authorization`；代理认证读 `Proxy-Authorization` | `auth.go:56`、`auth.go:105` |
| 失败状态码 | 普通 401，代理 407 | `auth.go:60`、`auth.go:109` |

## 5. 错误与重试语义

- 认证失败不返回 error 对象，而是直接写响应头 + `AbortWithStatus` 中止 handler 链；不存在重试。
- 空凭证表/空用户名在**启动期** `panic`（`auth.go:78`、`auth.go:81`），属于配置错误 fail-fast，而非请求期错误。
- base64 解码失败自然由比对不等兜底（请求头不是合法 Basic 串则常量时间比对返回 0）。
- 无重试、无退避；认证即一次性成败。

## 6. 并发细节

- 不创建 goroutine。
- `processAccounts` 在启动期把 `Accounts map` 转成 `authPairs` 切片；中间件闭包持有该切片，请求期只读遍历，**无锁**。
- `subtle.ConstantTimeCompare` 不依赖任何锁，纯内存字节比较。
- 用户名写入 `c.Keys` 由 `Context.mu sync.RWMutex` 保护（`context.go:286-294`）。
- context 传播：本中间件不设 deadline。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `auth.go` 全部：Basic/Proxy 认证中间件、凭证预计算、常量时间比对、用户名回写。

**Out-of-Scope（不在本仓库源码内）**
- `crypto/subtle`（`auth.go:8`）：常量时间比较，外部依赖。
- `encoding/base64`（`auth.go:9`）：凭证编码，外部依赖。
- `internal/bytesconv`（`auth.go:13`）：`StringToBytes` 零拷贝，见 internal-utils 叶子。
- `Context.Set/MustGet/AbortWithStatus/requestHeader` 实现在 `context.go`（context-object 叶子）。
- 本叶子不实现 token/session/JWT 认证，仅 HTTP Basic。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户通过 `RouterGroup.Use` 或路由级 handler 传入 `BasicAuth(accounts)`；注册时 `processAccounts` 预计算凭证。
- 本叶子 → 下游：认证成功后不调用 `c.Next()`（它作为中间件最后一环，后续 handler 由框架自动继续）；失败则 `AbortWithStatus` 切断链。
- 本叶子 → 相邻：写入 `c.Keys`（context-object 叶子）；使用 `internal/bytesconv`（internal-utils 叶子）做零拷贝。

## 9. 语言专项适配口径

- **安全敏感**：用 `crypto/subtle.ConstantTimeCompare` 替代 `==`，消除时序侧信道；这是 Go 标准库推荐的密码学常量时间比较做法。
- **预计算**：启动期把 `user:pass` 统一编码为 `Basic xxx`，使请求期只需对字符串做一次常量时间比对，避免每请求重复 base64。
- 无 goroutine/channel；纯同步中间件。
- 无 K8s 控制器模式。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| BasicAuth 中间件架构图 | `basic-auth-architecture.html` | architecture | showcase |
| JSON IR | `json/basic-auth-architecture.json` | — | — |

本叶子不补时序图：认证主路径为"取头→比对→成功写键/失败 401"的二分支线性流程，已在第 3 节文字化。
