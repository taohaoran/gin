# 响应渲染（render）域总览

> 本域包含以下叶子子系统；各叶子详情见对应文档。
> 源码基准：`github.com/gin-gonic/gin` master，commit `5c6a15f`（Go 1.26.0）。

## 1. 域职责

`render` 域负责**把 Go 数据对象渲染为 HTTP 响应**：统一 `Render` 接口（`Render(w)` + `WriteContentType(w)`），覆盖 JSON 六种变体、XML、YAML、TOML、MsgPack、Protobuf、BSON、纯文本、HTML 模板、原始字节、Reader 流、PDF 与 Redirect。`Context.Render` 编排状态码、Content-Type 幂等写入、响应体与错误处理。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| render-core | [render-core.md](render-core/render-core.md) | [架构图](render-core/render-core-architecture.html) | [时序图](render-core/render-core-sequence.html) | Render 接口、写响应协议、writeContentType |
| render-formats | [render-formats.md](render-formats/render-formats.md) | [架构图](render-formats/render-formats-architecture.html) | [数据流图](render-formats/render-formats-dataflow.html) | 各格式渲染器的内容类型与输出行为 |

## 3. 域级机制细节

- **两阶段写头**：`WriteContentType` 仅在响应头未设置 Content-Type 时写入，不覆盖业务自定义头。
- **无 body 状态短路**：204/304 等状态只写头不写体（`Context.Render` 判 `bodyAllowedForStatus`）。
- **渲染错误即中止**：`r.Render` 返 error 时 `c.Error(err)` + `c.Abort()`。
- **JSON 家族变体**：同一 `Data`，靠缩进/Secure 前缀/JSONP 回调/ASCII 转义/关闭 HTML 转义区分。
- **Redirect 空写头**：`WriteContentType` 为空实现，非法状态码直接 panic。

## 4. 域级图

![render 域架构图](render-architecture.html)

![render 域响应输出时序](render-sequence.html)

![render 域响应输出数据流](render-dataflow.html)
