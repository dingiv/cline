# Cline 源码阅读笔记

> Cline v3.79.0 - VS Code 自主编码代理扩展
> 约 32 万行 TypeScript，47 个 AI 提供商，26 个内置工具

## 笔记目录

| 主题        | 文件                                         | 核心内容                        |
| ----------- | -------------------------------------------- | ------------------------------- |
| 项目结构    | [project-structure.md](project-structure.md) | 目录布局、模块划分、代码规模    |
| 启动流程    | [startup-flow.md](startup-flow.md)           | 扩展激活、初始化、服务注册      |
| Agent 循环  | [agent-loop.md](agent-loop.md)               | 任务循环、递归请求、流式处理    |
| Prompt 构建 | [prompt-building.md](prompt-building.md)     | 变体系统、组件化、模板引擎      |
| 状态管理    | [state-management.md](state-management.md)   | StateManager、缓存策略、持久化  |

| 工具执行    | [tool-execution.md](tool-execution.md)       | 26 个工具、自动审批、Hook 机制  |
| 通信机制    | [communication.md](communication.md)         | ProtoBus/gRPC、前后端消息传递   |
| API 提供商  | [api-providers.md](api-providers.md)         | 47 个提供商、统一接口、流式响应 |

## 核心数据流（全局视角）

```
┌─────────────────────────────────────────────────────────┐
│                    用户输入                               │
│              (Webview React / CLI Ink)                   │
└──────────────────────┬──────────────────────────────────┘
                       │ postMessage / ProtoBus
                       ▼
┌─────────────────────────────────────────────────────────┐
│                  Controller                              │
│    [controller/index.ts](../src/core/controller/index.ts)│
│    • Task 生命周期管理（创建/取消/清除）                    │
│    • 设置更新、认证、MCP 服务                              │
│    • gRPC 请求路由（ServiceRegistry）                     │
└──────────────────────┬──────────────────────────────────┘
                       │ initTask() / startTask()
                       ▼
┌─────────────────────────────────────────────────────────┐
│                    Task                                  │
│    [task/index.ts](../src/core/task/index.ts) (~3300行)  │
│                                                          │
│  initiateTaskLoop()          ← 外层 while 循环            │
│    └── recursivelyMakeClineRequests()  ← 内层递归         │
│          ├── getSystemPrompt()     组装系统提示词          │
│          ├── api.createMessage()   调用 AI API（流式）     │
│          ├── parseAssistantMessageV2()  解析响应           │
│          └── presentAssistantMessage()  分发处理           │
│                ├── text → 显示到 UI                       │
│                └── tool_use → ToolExecutor                │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              ToolExecutor                                │
│    [ToolExecutor.ts](../src/core/task/ToolExecutor.ts)   │
│    → ToolExecutorCoordinator → 26个Handler               │
│    → 工具结果 → 作为下一轮 userMessage → 递归继续          │
└─────────────────────────────────────────────────────────┘
```
