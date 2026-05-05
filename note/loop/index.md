# Agent 循环

> - [task/index.ts](/src/core/task/index.ts) — Task 类，约 3300 行，整个代理循环引擎
> - [ToolExecutor.ts](/src/core/task/ToolExecutor.ts) — 工具执行入口
> - [assistant-message/index.ts](/src/core/assistant-message/index.ts) — 助手消息解析

Agent 循环的基本过程可以分为如下 4 步

+ 构建 prompt
+ 发起请求
+ 流式监听，实时解析
  + 如果是 tools_call，则执行调用，返回调用的结果
  + 如果是结束标识，则退出循环
  + 如果是生成的内容，则刷新 UI
  + 如果是空消息，说明是异常情况，记录错误，并退出
+ 保存上下文

## Task 类与双层循环结构
Agent 循环的能力通过 Task 的方法进行实现。

```typescript
class Task {
    // 核心子系统
    taskState: TaskState              // 当前任务的可变状态
    messageStateHandler               // 消息数组管理（API + UI）
    contextManager                    // 上下文窗口管理
    diffViewProvider / fileEditProvider // 文件编辑方式
    browserSession                    // 浏览器自动化
    commandExecutor                   // Shell 命令执行
    toolExecutor                      // 工具路由
    streamResponseHandler             // 流式响应处理
    checkpointManager                 // Git 检查点

    // 安全控制
    clineIgnoreController             // .clineignore
    commandPermissionController       // 命令权限
}
```

### 外层循环：initiateTaskLoop()

**位置**：[task/index.ts](/src/core/task/index.ts) ~line 1453

```typescript
async initiateTaskLoop(userContent: UserContent) {
    while (!taskState.abort) {
        // 调用内层递归
        const didEndLoop = await recursivelyMakeClineRequests(userContent)

        if (didEndLoop) break

        // 如果模型回复了文本但没有使用工具 → 推回提醒
        userContent = [formatResponse.noToolsUsed(consecutiveMistakeCount)]
        consecutiveMistakeCount++
    }
}
```

**作用**：确保 AI 必须通过工具或 `attempt_completion` 来推进任务，纯文本回复会被"打回"。

### 内层循环：recursivelyMakeClineRequests()

**位置**：[task/index.ts](/src/core/task/index.ts) ~line 2351

这是递归函数，每一轮代表一次完整的 API 调用 + 工具执行：

![](./loop.dio.svg)

```
recursivelyMakeClineRequests(userContent)
    │
    ├── 1. 检查错误计数（consecutiveMistakeCount 上限）
    │
    ├── 2. attemptApiRequest()  ← 异步生成器，返回 API 流
    │       ├── 等待 MCP 服务器连接
    │       ├── 构建 SystemPromptContext
    │       ├── getSystemPrompt() → 组装系统提示词
    │       ├── contextManager.getNewContextMessagesAndMetadata()
    │       │     └── 管理上下文窗口，必要时截断历史
    │       └── api.createMessage() → AsyncGenerator<ApiStreamChunk>
    │
    ├── 3. StreamChunkCoordinator 处理流式响应
    │       ├── reasoning 块 → 思考过程
    │       ├── text 块 → 文本内容
    │       ├── tool_calls 块 → 工具调用
    │       └── usage 块 → Token 用量统计
    │
    ├── 4. parseAssistantMessageV2() → 解析为 content blocks
    │
    ├── 5. presentAssistantMessage() → 逐块处理
    │       ├── text block → say("text") → 发送到 UI
    │       └── tool_use block → toolExecutor.executeTool()
    │
    ├── 6. 保存消息到磁盘
    │
    ├── 7. 如果使用了工具：
    │       收集工具结果 → 组装为新的 userContent
    │       → 递归调用自身 (return recursivelyMakeClineRequests(toolResults))
    │
    └── 8. 返回 true（应结束循环）或递归继续
```

## attemptApiRequest() 详解

**位置**：[task/index.ts](../src/core/task/index.ts) ~line 1865

这是一个异步生成器（`async function*`），核心逻辑：

```typescript
async function* attemptApiRequest(previousApiReqIndex?: number) {
    // 1. 等待 MCP 服务器就绪
    await waitForMcpConnections()

    // 2. 构建系统提示词上下文
    const promptContext = {
        cwd, ide, provider, model,
        mcpHub, skills, rules,
        browserSettings, mode,
        workspaceRoots, ...
    }

    // 3. 获取系统提示词（含工具定义）
    const { systemPrompt, nativeTools } = await getSystemPrompt(promptContext)

    // 4. 管理上下文窗口
    const { messages, metadata } = await contextManager.getNewContextMessages(...)

    // 5. 调用 API（流式）
    const stream = api.createMessage(systemPrompt, messages, tools)

    // 6. 重试逻辑
    //    - 首个 chunk 失败 → 自动重试最多 3 次（指数退避）
    //    - 上下文溢出 → 自动截断历史并重试 1 次

    yield* stream  // 向上层传递流式 chunks
}
```

## 消息解析：parseAssistantMessageV2()

[assistant-message/index.ts](../src/core/assistant-message/index.ts)

将模型的原始输出解析为结构化的 content blocks：

```typescript
type ContentBlock =
    | { type: "text"; content: string }
    | { type: "tool_use"; name: string; params: Record<string, string> }
```

解析过程需要处理：
- XML 格式的工具调用（`<read_file><path>...</path></read_file>`）
- 原生 JSON 工具调用（某些 API 直接返回结构化工具调用）
- 混合内容（文本 + 多个工具调用交错）

## 上下文窗口管理

[context/](../src/core/context/)

```
ContextManager
    ├── 维护 API 消息历史（apiConversationHistory）
    ├── 监控 token 使用量
    ├── 当接近上下文限制时：
    │     ├── 截断较早的消息
    │     └── 可选：使用 condense 工具压缩上下文
    └── 保留最近的工具调用结果（避免丢失关键上下文）
```

## 任务入口

```typescript
// 新任务
task.startTask(prompt: string, images?: string[], files?: string[])
    → 运行 TaskStart hooks
    → 构建用户内容
    → initiateTaskLoop(userContent)

// 恢复任务
task.resumeTaskFromHistory()
    → 加载保存的消息
    → 构建用户内容
    → initiateTaskLoop(userContent)
```
