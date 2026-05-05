# SSE (Server-Sent Events)

> SSE 是一个纯文本流式协议，只定义**帧格式**，不定义**内容格式**。
> JSON 是 Anthropic/OpenAI 等提供商自己的约定，不是 SSE 规范的一部分。

## SSE 规范

极其简单，就一个普通 HTTP 响应，`Content-Type: text/event-stream`，连接不关闭，持续推送文本：

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: content_block_start\n
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":"Hello"}}\n
\n                                          ← 空行 = 一个事件结束
event: content_block_delta\n
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}\n
\n
```

规则：
- 每个事件由多个 `field: value` 行组成
- **空行**分隔不同事件
- `data:` 字段是纯文本，内容格式不限（JSON / XML / 纯文本都行）
- `event:`、`id:`、`retry:` 可选

## SSE 规范 vs 提供商约定

```
SSE 规范定义的:              提供商自己定义的:
━━━━━━━━━━━━━━             ━━━━━━━━━━━━━━━━━━━━━
event: xxx                 {"type":"message_start",
data: xxx  ← 纯文本             "message":{"usage":{...}}}
id: xxx                          ↑ JSON 结构是各家自己设计的
retry: xxx
```

## Cline 中的位置

Cline 不直接处理 SSE 文本。SDK 负责解析：

```
HTTP SSE 文本流
    │
    ▼
@anthropic-ai/sdk / openai SDK
    │  解析 event:/data: 行，JSON.parse data 内容
    ▼
AsyncIterable (JS 对象流)
    │
    ▼
Cline ApiHandler
    │  把 SDK 事件映射为统一的 ApiStreamChunk
    ▼
AsyncGenerator<ApiStreamChunk>
    │
    ▼
Task 循环消费
```

Cline 拿到的已经是解析好的 JS 对象，不关心底层传输协议是 SSE 还是 WebSocket。
