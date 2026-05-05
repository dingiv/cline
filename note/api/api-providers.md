# API 提供商

> - [api/index.ts](/src/core/api/index.ts) — `buildApiHandler()` 工厂函数 + `ApiHandler` 接口
> - [api/providers/](/src/core/api/providers/) — 47 个提供商的具体实现
> - [shared/api.ts](/src/shared/api.ts) — `ApiProvider` 联合类型、`ModelInfo` 接口、模型定义
> - [providers.json](/src/shared/providers/providers.json) — 提供商列表
> - [model-utils.ts](/src/utils/model-utils.ts) — 模型工具函数

各个 API 提供商的模型服务的接口其实不尽相同，但是 openai 有先发优势，后来的厂商为了夺取 openai 的用户，干脆就直接兼容 openai 的 API 接口。目前，市面上，存在最多的就是 openai，anthropic，gemini 三家的 API 风格。

```py
from openai import OpenAI

client = OpenAI()

stream = client.responses.create(
    model="gpt-5.2",
    input="Write a one-sentence bedtime story about a unicorn.",
    stream=True,
)

for event in stream:
    print(event)
```

## ApiHandler 统一接口

```typescript
interface ApiHandler {
    createMessage(
        systemPrompt: string,
        messages: ClineStorageMessage[],
        tools?: ClineTool[],
        useResponseApi?: boolean
    ): ApiStream

    getModel(): { id: string; info: ModelInfo }
    getApiStreamUsage?(): Promise<ApiStreamUsageChunk>
    abort?(): void
}
```

### ApiStream（流式响应）
[stream.ts](../src/core/api/transform/stream.ts)

```typescript
type ApiStream = AsyncGenerator<ApiStreamChunk>

type ApiStreamChunk =
    | { type: "text"; text: string }
    | { type: "reasoning"; text: string }
    | { type: "tool_calls"; toolCalls: ToolCall[] }
    | { type: "usage"; inputTokens: number; outputTokens: number; cacheRead?: number; cacheWrite?: number }
```

## 47 个提供商

### 主流提供商

| 提供商          | 工厂分支            | 特点                             |
| --------------- | ------------------- | -------------------------------- |
| `anthropic`     | AnthropicHandler    | Claude 系列，支持思考、缓存      |
| `openai-native` | OpenAINativeHandler | GPT 系列（Chat Completions API） |
| `openai-codex`  | OpenAICodexHandler  | Codex（Responses API）           |
| `openrouter`    | OpenRouterHandler   | 聚合多模型                       |
| `bedrock`       | AwsBedrockHandler   | AWS Bedrock                      |
| `vertex`        | VertexHandler       | Google Vertex AI                 |
| `gemini`        | GeminiHandler       | Google Gemini                    |
| `deepseek`      | DeepSeekHandler     | DeepSeek                         |
| `ollama`        | OllamaHandler       | 本地模型                         |
| `lmstudio`      | LmStudioHandler     | 本地模型                         |
| `cline`         | ClineHandler        | Cline 自有服务                   |

### 其他提供商

`mistral`, `vscode-lm`, `litellm`, `moonshot`, `huggingface`, `nebius`, `asksage`, `xai`, `sambanova`, `cerebras`, `groq`, `baseten`, `fireworks`, `together`, `qwen`, `qwen-code`, `doubao`, `requesty`, `sapaicore`, `claude-code`, `huawei-cloud-maas`, `dify`, `vercel-ai-gateway`, `zai`, `oca`, `aihubmix`, `minimax`, `hicap`, `nousresearch`, `wandb`

## 提供商选择逻辑

[api/index.ts](../src/core/api/index.ts)

```typescript
function buildApiHandler(config: ApiConfiguration): ApiHandler {
    const provider = config.planModeApiProvider ?? config.apiProvider

    switch (provider) {
        case "anthropic": return new AnthropicHandler(config)
        case "openai-native": return new OpenAINativeHandler(config)
        case "openrouter": return new OpenRouterHandler(config)
        // ... 47 个 case
        default: return new AnthropicHandler(config)  // 默认
    }
}
```

## 适配器层

[adapters/index.ts](../src/core/api/adapters/index.ts)

不同提供商对工具调用的格式支持不同，适配器负责转换：

```typescript
transformToolCallMessages(messages, tools, providerSupports)
    ├── apply_patch ↔ write_to_file/replace_in_file  格式转换
    ├── 原生 JSON 工具 ↔ XML 工具                     格式转换
    └── 根据提供商能力自动选择最佳格式
```
