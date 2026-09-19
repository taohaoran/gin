# 响应渲染（response-render）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin`，commit `5c6a15f`。

## 1. 域职责

本域负责把 **Go 数据结构**渲染成 HTTP 响应（JSON/XML/YAML/TOML/ProtoBuf/MsgPack/BSON/HTML/文本/二进制/重定向/流式）。
核心代码路径：`Context.Render(code, r)`（`context.go:1202`）→ 先 `Status(code)` → 空 body 状态码短路 → 调 `Render.Render(w)` 与 `WriteContentType` → 出错 `c.Error+c.Abort`。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 | 职责一句话 |
|------|------|--------|--------|-----------|
| render-interface | [render-interface.md](render-interface/render-interface.md) | [架构图](render-interface/render-interface-architecture.html) | [时序图](render-interface/render-interface-sequence.html) | Render 两阶段接口、Content-Type 幂等写入、JSON 六种变体 |
| markup-renderers | [markup-renderers.md](markup-renderers/markup-renderers.md) | [架构图](markup-renderers/markup-renderers-architecture.html) | — | XML/YAML/TOML/ProtoBuf/MsgPack/BSON/Data/String/PDF 渲染器 |
| html-redirect-reader | [html-redirect-reader.md](html-redirect-reader/html-redirect-reader.md) | [架构图](html-redirect-reader/html-redirect-reader-architecture.html) | — | HTML Debug/Production 双模板、Redirect 重定向、Reader 流式透传 |

## 3. 域级机制细节

- **两阶段提交**：`WriteContentType`（幂等，不覆盖已有值）与 `Render`（写 body）分离；空 body 状态码只写 Content-Type。
- **错误统一收口**：渲染 error 压入 `c.Errors` 并 `Abort`。
- **JSON 统一编码后端**：所有 JSON 变体经 `codec/json.API`，可经 build tag 切 sonic/jsoniter/go-json（见 codec-json 域）。
- **HTML 双轨**：DebugMode 每次请求重新解析模板，ReleaseMode 用预编译 `HTMLProduction`。

## 4. 域级图

本域不单独绘制域级架构图；render-interface 的时序图已覆盖 `c.JSON → Render → 写 Content-Type → 写 body` 主路径。
