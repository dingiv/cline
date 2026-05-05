# AI 后端交互机制

> Cline 通过统一的 `ApiHandler` 接口封装 47 个 AI 提供商的差异，
> 以 `AsyncGenerator<ApiStreamChunk>` 流式响应为桥梁，连接 Task 循环与底层 SDK。

## 整体架构
![](./loop.dio.svg)


```
┌─────────────────────────────────────────────────────────────┐
│  Task 循环 (task/index.ts ~3300行)                          │
│    systemPrompt + conversationHistory + tools                │
│         │                                                   │
│         ▼                                                   │
│  api.createMessage()  ←── ApiHandler 统一接口               │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────────────────────────────┐                   │
│  │  Provider Handler (47个实现)          │                   │
│  │  1. 消息格式转换 (sanitize)           │                   │
│  │  2. 构建请求体 (model/tools/thinking) │                   │
│  │  3. @withRetry() 重试装饰器           │                   │
│  │  4. SDK 调用 → 原始 stream            │                   │
│  │  5. 事件映射 → ApiStreamChunk         │                   │
│  └──────────────────┬───────────────────┘                   │
│                     │                                       │
│                     ▼                                       │
│  StreamChunkCoordinator                                     │
│    ├── usage → onUsageChunk (实时更新 token，不阻塞)          │
│    └── text/reasoning/tool_calls → queue                    │
│                     │                                       │
│                     ▼                                       │
│  Task 主循环消费                                             │
│    ├── text → 累积 → presentAssistantMessage() → UI         │
│    ├── reasoning → UI 折叠展示                               │
│    └── tool_calls → ToolExecutor → 结果 → 下一轮递归          │
└─────────────────────────────────────────────────────────────┘
```

## 1. 统一接口 — `ApiHandler`

**文件**: `src/core/api/index.ts`

所有 47 个提供商都实现同一个接口：

```typescript
interface ApiHandler {
  createMessage(
    systemPrompt: string,
    messages: ClineStorageMessage[],
    tools?: ClineTool[],
    useResponseApi?: boolean
  ): ApiStream
  getModel(): ApiHandlerModel
  getApiStreamUsage?(): Promise<ApiStreamUsageChunk | undefined>
  abort?(): void
}
```

**Handler 工厂** — `createHandlerForProvider()`:
- 巨大的 switch 语句（47 个 case）
- 根据 provider 字符串 + mode (plan/act) 创建对应 Handler
- plan/act 模式可使用不同模型（如 plan 用便宜模型，act 用强模型）

**关键 Handler 实现**（都位于 `src/core/api/providers/`）:

| Handler | 文件 | SDK |
|---------|------|-----|
| AnthropicHandler | `anthropic.ts` | `@anthropic-ai/sdk` |
| OpenAiHandler | `openai.ts` | `openai` |
| OpenAiNativeHandler | `openai-native.ts` | `openai` (Responses API) |
| GeminiHandler | `gemini.ts` | `@google/generative-ai` |
| VertexHandler | `vertex.ts` | `@anthropic-ai/sdk` (Vertex 端点) |
| OllamaHandler | `ollama.ts` | 原生 fetch |
| OpenRouterHandler | `openrouter.ts` | 原生 fetch |

## 2. 流式响应类型 — `ApiStream`

**文件**: `src/core/api/transform/stream.ts`

```typescript
type ApiStream = AsyncGenerator<ApiStreamChunk> & { id?: string }
type ApiStreamChunk =
  | ApiStreamTextChunk      // 模型文本输出
  | ApiStreamThinkingChunk   // 思维链/推理
  | ApiStreamUsageChunk      // token 用量统计
  | ApiStreamToolCallsChunk  // 原生工具调用
```

四种 Chunk 详细结构：

```typescript
// 文本
{ type: "text", text: string, id?: string, signature?: string }

// 推理（Claude thinking / Gemini thinking）
{ type: "reasoning", reasoning: string, signature?: string,
  details?: unknown, redacted_data?: string, id?: string }

// 用量
{ type: "usage", inputTokens: number, outputTokens: number,
  cacheWriteTokens?: number, cacheReadTokens?: number,
  thoughtsTokenCount?: number, totalCost?: number, id?: string }

// 工具调用（OpenAI 兼容格式，Anthropic 也会转成这个格式）
{ type: "tool_calls", tool_call: {
    call_id?: string,
    function: { id?: string, name?: string, arguments?: any }
  }, id?: string, signature?: string }
```

## 3. 提供商适配 — 以 Anthropic 为例

**文件**: `src/core/api/providers/anthropic.ts`

### 3.1 客户端初始化

```typescript
private ensureClient(): Anthropic {
  if (!this.client) {
    this.client = new Anthropic({
      apiKey: this.options.apiKey,
      baseURL: this.options.anthropicBaseUrl || undefined,
      defaultHeaders: buildExternalBasicHeaders(),
      fetch,  // ← 关键：使用代理感知的 fetch（支持企业代理）
    })
  }
  return this.client
}
```

### 3.2 消息格式转换

不同提供商的消息格式不同，每个 Handler 在发请求前做转换：
- **Anthropic**: `sanitizeAnthropicMessages(messages, true)` — 处理图片、缓存等
- **OpenAI**: `convertToOpenAiMessages()` — 转换角色名、图片格式
- **Gemini**: `convertAnthropicMessageToGemini()` — 完全不同的多模态格式

### 3.3 请求体构建

```typescript
const requestBody = {
  model: modelId,
  thinking: thinkingConfig,           // 推理配置（adaptive/budgeted）
  max_tokens: model.info.maxTokens,
  temperature: reasoningOn ? undefined : 0,
  system: [{ text: systemPrompt, type: "text", cache_control: { type: "ephemeral" } }],
  messages: anthropicMessages,
  tools: nativeToolsOn ? tools : undefined,
  tool_choice: nativeToolsOn && !thinkingEnabled ? { type: "any" } : undefined,
  stream: true,
}
```

关键点：
- **Prompt Cache**: system prompt 设置 `cache_control` breakpoint，新任务可复用缓存
- **Thinking**: 支持 `adaptive`（Opus 4.5+）和 `budgeted`（旧版）两种模式
- **Temperature**: 开启 thinking 时必须设为 undefined（API 限制）
- **tool_choice**: thinking 开启时不能强制 tool_use（API 限制）

### 3.4 原始事件 → 统一 Chunk 映射

Anthropic SDK 的 `client.messages.create({ stream: true })` 返回的事件流：

```
message_start          → { type: "usage", inputTokens, cacheWriteTokens, cacheReadTokens }
message_delta          → { type: "usage", outputTokens }
content_block_start
  ├── type: "text"     → { type: "text", text: "..." }
  ├── type: "thinking"  → { type: "reasoning", reasoning: "...", signature }
  ├── type: "redacted_thinking" → { type: "reasoning", reasoning: "[Redacted]", redacted_data }
  └── type: "tool_use" → 记录 id/name，等待后续 delta
content_block_delta
  ├── type: "text_delta"       → { type: "text", text: "..." }
  ├── type: "thinking_delta"   → { type: "reasoning", reasoning: "..." }
  ├── type: "signature_delta"  → { type: "reasoning", reasoning: "", signature }
  └── type: "input_json_delta" → { type: "tool_calls", tool_call: { call_id, function: { name, arguments } } }
content_block_stop      → 清空 lastStartedToolCall
```

注意：Anthropic 的 tool_use 事件分两步到达（start 给 id/name，delta 给参数），
Handler 内部用 `lastStartedToolCall` 状态变量拼接，最终转成 OpenAI 兼容格式输出。

## 4. 重试机制

**文件**: `src/core/api/retry.ts`

`@withRetry()` 是一个方法装饰器，包裹每个 Handler 的 `createMessage()`：

```typescript
@withRetry()
async *createMessage(...): ApiStream { ... }
```

重试策略：
- **默认**: 最多 3 次，基础延迟 1s，最大延迟 10s
- **429 限流**: 读取 `retry-after` / `x-ratelimit-reset` / `ratelimit-reset` header 精确等待
- **无 header**: 指数退避 `min(10s, 1s × 2^attempt)` = 1s → 2s → 4s
- **回调通知**: `onRetryAttempt(attempt, maxRetries, delay, error)` 通知 UI 显示重试状态
- **错误类型**: 默认只重试 429 和 `RetriableError`，`retryAllErrors: true` 时重试所有错误

## 5. 流消费与分发

### 5.1 StreamChunkCoordinator

**文件**: `src/core/task/StreamChunkCoordinator.ts`

这是一个**生产者-消费者**模式的协调器，解决一个关键问题：
usage 更新需要实时反映（token 计数），但内容处理可能阻塞（等工具执行、等用户输入）。

```
API Stream (AsyncIterator)
        │
        ▼
  startPump() (后台异步循环)
    ├── type === "usage" → 立即调用 onUsageChunk()，不入队
    └── type !== "usage" → push 到 queue，唤醒 waiter
        │
        ▼
  nextChunk() (Task 主循环调用)
    ├── queue 有数据 → shift 返回
    ├── completed → return undefined (流结束)
    └── 都没有 → await waitForData() (挂起直到有新数据)
```

核心实现是一个 promise-based 的等待机制：
```typescript
private async waitForData() {
  if (this.queue.length > 0 || this.completed || this.readError) return
  await new Promise<void>((resolve) => { this.waiterResolve = resolve })
}
```

### 5.2 Task 主循环消费

**文件**: `src/core/task/index.ts`

```typescript
// 1. 创建 stream
const stream = this.api.createMessage(systemPrompt, messages, tools)

// 2. 用 Coordinator 包裹
const coordinator = new StreamChunkCoordinator(stream, {
  onUsageChunk: (chunk) => { /* 实时更新 token/cost */ }
})

// 3. 逐 chunk 消费
while (true) {
  const chunk = await coordinator.nextChunk()
  if (!chunk) break  // 流结束

  switch (chunk.type) {
    case "text":
      // 累积文本 → presentAssistantMessage() → UI 实时显示
      break
    case "reasoning":
      // 累积推理内容 → UI 折叠展示
      break
    case "tool_calls":
      // ToolUseHandler 累积参数 JSON
      // 参数完整时 → ToolExecutor.executeTool()
      break
  }
}
```

### 5.3 工具调用 → 递归

```
tool_calls chunk 到达
  │
  ▼
ToolUseHandler.processToolUseDelta()
  ├── 累积 partial JSON 参数
  └── 参数完整 → processNativeToolCalls()
        │
        ▼
ToolExecutor.executeTool(toolUseBlock)
  │
  ▼
ToolExecutorCoordinator → 对应 Handler (如 ExecuteCommandToolHandler)
  │
  ▼
工具执行结果 → 格式化为 userMessage
  │
  ▼
递归: recursivelyMakeClineRequests() ← 带着工具结果继续下一轮
```

## 核心设计思想

1. **接口统一**: `ApiHandler` + `ApiStreamChunk` 把 47 个提供商差异封装在 Handler 内部
2. **流式驱动**: 全链路基于 `AsyncGenerator`，从 SDK 原始事件到 UI 渲染都是增量处理
3. **关注点分离**: StreamChunkCoordinator 把 usage（实时统计）和 content（可能阻塞）分开处理
4. **装饰器增强**: `@withRetry()` 透明地给所有 Handler 加上重试能力，不侵入业务逻辑
5. **格式归一**: 所有提供商的 tool_call 最终都映射为 OpenAI 兼容格式，下游只需处理一种结构

## 扩展新提供商的步骤

1. 在 `src/core/api/providers/` 创建新 Handler，实现 `ApiHandler` 接口
2. 在 `createHandlerForProvider()` switch 中添加 case
3. 在 `src/shared/api.ts` 添加 provider 类型、模型定义
4. 在 `src/shared/providers/providers.json` 添加下拉选项
5. 实现 SDK 事件 → `ApiStreamChunk` 的映射
6. 如需 XML 工具（非原生），确保 prompt 中有对应工具定义
