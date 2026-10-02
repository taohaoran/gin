# gin 项目探查事实（archify 分析只读探查产出）

## 1. 身份
- 模块: `github.com/gin-gonic/gin`（go.mod: `go 1.26.0`）
- Git: master @ `5c6a15f`（1.12.0 发布期；近期提交含 QUERY 方法快捷 RFC 10008、RunQUIC、可插拔 JSON codec、x/crypto 补丁）
- LICENSE: MIT；README 定位：高性能 HTTP Web 框架（Martini-like API，httprouter 血缘，零分配路由）

## 2. 规模
- 口径: `find -name '*.go' -not -path '*/testdata/*'`（含测试）→ 98 个文件、23,928 行
- 核心非测试文件体积（约）: context.go 48KB / gin.go 27KB / tree.go 24KB / routergroup.go 9.7KB / logger.go 8KB / recovery.go 5.8KB / path.go 4.8KB / errors.go 3.9KB / auth.go 3.8KB / response_writer.go 3.4KB / mode.go 2.4KB / debug.go 3KB / utils.go 4.2KB / fs.go 1.4KB / version.go 0.2KB / deprecated.go 0.7KB
- 无 `cmd/`（纯库模块，无二进制入口）；`ginS/` 为示例服务器（gins.go + gins_test.go + README.md，非库代码）
- 无 `AGENTS.md`

## 3. 结构
- 顶层单包 `package gin`（核心文件均在仓库根）
- 子包: `binding/`（30 文件：binding.go、form_mapping.go、multipart_form_mapping.go、default_validator.go、uri.go + json/xml/yaml/toml/msgpack/protobuf/bson/plain/header/query/form 等格式实现）
- `render/`（19 文件：render.go + json/xml/yaml/toml/msgpack/protobuf/bson/text/html/redirect/data/reader/pdf 渲染器）
- `internal/bytesconv`（StringToBytes 等零拷贝转换）、`internal/fs`（路径清理）
- `codec/json/`（可插拔 JSON 编解码: api.go 接口 + go_json.go / json.go(标准库) / jsoniter.go / sonic.go 四实现）
- `docs/`（docs/doc.md 用户文档）、`testdata/`、`examples/`、`.github/`

## 4. 功能清单（README + 近期特性）
- 零分配路由（基数树）、高性能、中间件系统、内置 recovery 防崩溃、JSON 绑定与校验、路由分组、集中错误管理、内置多格式渲染、可扩展生态
- 近期: QUERY 方法快捷（RFC 10008）、RunQUIC（quic-go）、可插拔 JSON codec、HTML renderer 缺失 panic 处理

## 5. 约束
- 项目根已存在 `docs/` → **输出根 = `<repo>/docs_archify/architecture/`**（先建 docs_archify/ 再建 architecture/）
- `docs_archify/architecture/` 当前不存在产物 → 首次分析，无需 improve 对比目录（如执行中发现已存在产物，按基线/对比规则处理）
- 输出一律简体中文；文件名/目录名英文短横线

## 6. 主语言判定
- Go（读 `references/language-go.md`；Go 平台类图侧重: informer 事件管道→dataflow、reconciler 状态机→lifecycle、并发通信→sequence）
