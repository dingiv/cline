# 通信机制

## 核心设计：ProtoBus

Cline 不直接使用 `postMessage`/`addEventListener`，而是构建了一个 **gRPC 风格的 RPC 框架**（ProtoBus）：

```
┌──────────────────┐     postMessage      ┌──────────────────┐
│   Webview (React) │  ──────────────────► │   Extension      │
│                   │                      │   (Node.js)      │
│  ServiceClient    │  grpc_request        │  ServiceRegistry │
│  (自动生成)        │                      │  (handler路由)    │
│                   │  ◄────────────────── │                  │
│                   │  grpc_response       │                  │
└──────────────────┘                      └──────────────────┘
```

## 消息协议

### Webview → Extension

```typescript
// src/shared/WebviewMessage.ts
type WebviewMessage = {
    type: "grpc_request"
    grpc_request: {
        service: string      // 服务名，如 "TaskService"
        method: string       // 方法名，如 "createTask"
        message: Uint8Array  // 序列化的 Protobuf 消息
        request_id: string   // UUID，用于关联响应
        is_streaming: boolean // 是否为流式调用
    }
} | {
    type: "grpc_request_cancel"
    grpc_request_cancel: {
        request_id: string   // 要取消的请求 ID
    }
}
```

### Extension → Webview

```typescript
// src/shared/ExtensionMessage.ts
type ExtensionMessage = {
    type: "grpc_response"
    grpc_response: {
        message: Uint8Array       // 序列化的 Protobuf 响应
        request_id: string        // 对应请求的 ID
        error?: string            // 错误信息
        is_streaming?: boolean    // 是否为流式响应的一部分
        sequence_number?: number  // 流式响应的序列号
    }
}
```

## 请求处理管线

```
1. VscodeWebviewProvider.handleWebviewMessage()
       │ VscodeWebviewProvider.ts ~line 161
       │ 接收 postMessage
       ▼
2. handleGrpcRequest()
       │ grpc-handler.ts
       │ 根据 is_streaming 选择处理方式
       ▼
3a. handleUnaryRequest()           3b. handleStreamingRequest()
    │ 一次请求-响应                    │ 持续推送响应
    ▼                                 ▼
4. ServiceRegistry.lookup(service, method)
       │ grpc-service.ts
       │ 路由到注册的 handler 函数
       ▼
5. handler(context, request) → Response
       │ 执行业务逻辑
       ▼
6. 序列化响应 → postMessage 回 Webview
```

**关键文件**：
- [VscodeWebviewProvider.ts](../src/hosts/vscode/VscodeWebviewProvider.ts) — 消息接收入口
- [grpc-handler.ts](../src/core/controller/grpc-handler.ts) — gRPC 请求分发
- [grpc-service.ts](../src/core/controller/grpc-service.ts) — 服务注册表

## 13 个 Service Client

[grpc-client.ts](../webview-ui/src/services/grpc-client.ts)（自动生成）

| Client | 服务 | 主要方法 |
|--------|------|----------|
| `TaskServiceClient` | 任务管理 | createTask, cancelTask, getTaskHistory, askResponse |
| `StateServiceClient` | 状态订阅 | subscribeToState（流式）, updateSetting |
| `UiServiceClient` | UI 事件 | subscribeToPartialMessage（流式）, clickButton |
| `ModelsServiceClient` | 模型管理 | refreshModels, updateApiConfiguration |
| `AccountServiceClient` | 账户认证 | login, logout, subscribeToAuthCallback（流式） |
| `BrowserServiceClient` | 浏览器 | connectToBrowser, discoverBrowser |
| `CommandsServiceClient` | 上下文命令 | addCodeToContext, fixWithCline |
| `FileServiceClient` | 文件操作 | openFile, searchFiles, saveRule |
| `McpServiceClient` | MCP 服务 | addServer, deleteServer, getMarketplaceCatalog |
| `CheckpointsServiceClient` | 检查点 | getDiff, restoreCheckpoint |
| `SlashServiceClient` | 斜杠命令 | executeSlashCommand |
| `WebServiceClient` | Web 操作 | checkUrl, openUrl |
| `WorktreeServiceClient` | Worktree | createWorktree, deleteWorktree |

## 两种调用模式

### 一元调用（Unary）

```typescript
// Webview 端调用
const response = await TaskServiceClient.createTask(
    CreateTaskRequest.create({ text: "帮我重构代码" })
)
// 等待单个响应
```

### 流式订阅（Streaming）

```typescript
// Webview 端订阅状态更新
const unsubscribe = StateServiceClient.subscribeToState(
    EmptyRequest.create(),
    {
        onResponse: (state) => {
            // 每次状态变化都触发
            setState(JSON.parse(state.stateJson))
        },
        onError: (error) => { /* 错误处理 */ },
        onComplete: () => { /* 流结束 */ }
    }
)
// 取消订阅
unsubscribe()
```

## 请求生命周期管理

[grpc-request-registry.ts](../src/core/controller/grpc-request-registry.ts)

```typescript
class GrpcRequestRegistry {
    // 跟踪所有活跃请求
    private activeRequests: Map<string, AbortController>

    // 注册新请求
    register(requestId: string): AbortController

    // 取消请求
    cancel(requestId: string): void

    // 清理完成的请求
    cleanup(requestId: string): void
}
```

## Proto 定义 → 代码生成

```
proto/cline/*.proto           ← 源 .proto 文件
        │
        │ npm run protos
        ▼
src/shared/proto/cline/*.ts   ← 生成的 TypeScript 类型
src/generated/grpc-js/        ← 服务端 stub
src/generated/nice-grpc/      ← 客户端 stub
src/generated/hosts/          ← Host handler
webview-ui/src/services/grpc-client.ts  ← Webview Client（由 scripts/generate-protobus-setup.mjs 生成）
```

## 平台适配

[platform.config.ts](../webview-ui/src/config/platform.config.ts)

通过编译时常量 `__PLATFORM__` 选择平台：

| 平台 | postMessage 方式 | 消息编码 |
|------|-----------------|---------|
| VS Code | `vscode.postMessage()` | none（直接传递） |
| Standalone | `window.standalonePostMessage()` | JSON（跨边界序列化） |

## Host Bridge 模式

Extension 需要调用 IDE 功能时，通过 Host Bridge 抽象：

```typescript
interface HostBridge {
    diff: DiffServiceClientInterface     // 打开 Diff 编辑器
    env: EnvServiceClientInterface       // 剪贴板、版本、遥测
    window: WindowServiceClientInterface // 显示信息/错误框
    workspace: WorkspaceServiceClientInterface // 工作区路径
}
```

- VS Code：[hostbridge/](../src/hosts/vscode/hostbridge/) — 调用真实 VS Code API
- CLI：[cli/src/controllers/index.ts](../cli/src/controllers/index.ts) — 终端版本的 stub 实现
