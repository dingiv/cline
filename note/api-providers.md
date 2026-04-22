# API 提供商

## 核心文件

- [api/index.ts](../src/core/api/index.ts) — `buildApiHandler()` 工厂函数 + `ApiHandler` 接口
- [api/providers/](../src/core/api/providers/) — 47 个提供商的具体实现
- [shared/api.ts](../src/shared/api.ts) — `ApiProvider` 联合类型、`ModelInfo` 接口、模型定义
- [providers.json](../src/shared/providers/providers.json) — 提供商列表
- [model-utils.ts](../src/utils/model-utils.ts) — 模型工具函数

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

| 提供商 | 工厂分支 | 特点 |
|--------|---------|------|
| `anthropic` | AnthropicHandler | Claude 系列，支持思考、缓存 |
| `openai-native` | OpenAINativeHandler | GPT 系列（Chat Completions API） |
| `openai-codex` | OpenAICodexHandler | Codex（Responses API） |
| `openrouter` | OpenRouterHandler | 聚合多模型 |
| `bedrock` | AwsBedrockHandler | AWS Bedrock |
| `vertex` | VertexHandler | Google Vertex AI |
| `gemini` | GeminiHandler | Google Gemini |
| `deepseek` | DeepSeekHandler | DeepSeek |
| `ollama` | OllamaHandler | 本地模型 |
| `lmstudio` | LmStudioHandler | 本地模型 |
| `cline` | ClineHandler | Cline 自有服务 |

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

## Responses API 提供商

OpenAI Codex 等使用 Responses API 的提供商需要原生工具调用。

### 关键判断

```typescript
// src/utils/model-utils.ts
function isNextGenModelProvider(provider: string): boolean {
    return ["openai-codex", "openai-native", ...].includes(provider)
}

// src/shared/api.ts
ModelInfo {
    apiFormat?: ApiFormat.OPENAI_RESPONSES  // 标记需要原生工具
}
```

### 自动启用原生工具

```typescript
// src/core/task/index.ts
if (model.info.apiFormat === ApiFormat.OPENAI_RESPONSES) {
    enableNativeToolCalls = true  // 强制启用，忽略用户设置
}
```

### 添加 Responses API 提供商清单

1. `isNextGenModelProvider()` 添加提供商
2. 所有使用 Responses API 的模型设置 `apiFormat: ApiFormat.OPENAI_RESPONSES`
3. 系统自动处理后续

## 添加新提供商的完整步骤

### 1. 定义类型

[shared/api.ts](../src/shared/api.ts)

```typescript
type ApiProvider = "..." | "new-provider" | "..."
const newProviderModels: Record<string, ModelInfo> = { ... }
const newProviderDefaultModelId = "model-id"
```

### 2. Proto 枚举（三处必须同步）

| 文件 | 操作 |
|------|------|
| [models.proto](../proto/cline/models.proto) | ApiProvider 枚举添加 NEW_PROVIDER = N |
| [api-configuration-conversion.ts](../src/shared/proto-conversions/models/api-configuration-conversion.ts) | 两个转换函数都添加映射 |

### 3. 实现 Handler

| 文件 | 操作 |
|------|------|
| [providers/new-provider.ts](../src/core/api/providers/) | 实现 ApiHandler 接口 |
| [api/index.ts](../src/core/api/index.ts) | buildApiHandler 添加 case |

### 4. UI 配置

| 文件 | 操作 |
|------|------|
| [providers.json](../src/shared/providers/providers.json) | 提供商下拉列表 |
| [providerUtils.ts](../webview-ui/src/components/settings/utils/providerUtils.ts) | getModelsForProvider + normalizeApiConfiguration |
| [ApiOptions.tsx](../webview-ui/src/components/settings/ApiOptions.tsx) | 渲染提供商组件 |
| [validate.ts](../webview-ui/src/utils/validate.ts) | 验证逻辑 |

### 5. CLI 支持

| 文件 | 操作 |
|------|------|
| [ModelPicker.tsx](../cli/src/components/ModelPicker.tsx) | providerModels 映射 |

## 模型信息结构

```typescript
interface ModelInfo {
    maxTokens: number                  // 最大输出 token
    contextWindow: number              // 上下文窗口大小
    supportsImages?: boolean           // 是否支持图片
    supportsComputerUse?: boolean      // 是否支持计算机操作
    supportsPromptCache?: boolean      // 是否支持提示缓存
    inputPrice?: number                // 输入价格（per million tokens）
    outputPrice?: number               // 输出价格
    cacheWritesPrice?: number          // 缓存写入价格
    cacheReadsPrice?: number           // 缓存读取价格
    description?: string               // 模型描述
    thinking?: boolean                 // 是否支持思考
    reasoningEffort?: boolean          // 是否支持推理力度调节
    apiFormat?: ApiFormat              // API 格式（OpenAI Chat/Responses）
}
```

## 网络请求规范

所有扩展代码中的网络请求必须使用代理感知的封装：

```typescript
// 不要用全局 fetch
import { fetch } from '@/shared/net'              // ✅ 代理感知
const response = await fetch('https://...')

// 不要用默认 axios
import { getAxiosSettings } from '@/shared/net'   // ✅ 代理感知
const response = await axios.get(url, { ...getAxiosSettings() })

// 第三方客户端必须传入 fetch
import { fetch } from '@/shared/net'              // ✅
const client = new OpenAI({ fetch })
```
