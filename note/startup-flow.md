# 启动流程

## 总览

Cline 的启动分为 5 个阶段，从 VS Code 激活到准备好接受用户指令：

```
extension.ts:activate()
    │
    ├── 1. setupHostProvider()          ← 注册平台工厂函数
    ├── 2. cleanupLegacyStorage()       ← 迁移旧版数据
    ├── 3. exportToSharedFiles()        ← 导出共享状态文件
    ├── 4. common.ts: initialize()      ← 核心初始化
    └── 5. registerVSCodeCommands()     ← 注册命令/菜单/URI
```

## 阶段 1：HostProvider 注入

[host-provider.ts](../src/hosts/host-provider.ts) 是一个单例依赖注入容器，抽象了平台差异（VS Code / CLI / JetBrains）。

```typescript
// src/hosts/host-provider.ts
class HostProvider {
    // 注册的工厂函数
    private factories = {
        createWebviewProvider,    // Webview 创建
        createDiffServiceClient,  // Diff 编辑器
        createEnvServiceClient,   // 环境信息/剪贴板
        createWindowServiceClient, // 窗口操作
        createWorkspaceServiceClient, // 工作区操作
        createTerminalManager,    // 终端管理
    }
}
```

VS Code 在 [extension.ts](../src/extension.ts) 中注入 VS Code 特定实现；CLI 在 [cli/src/controllers/index.ts](../cli/src/controllers/index.ts) 中注入终端版实现。

## 阶段 2-3：存储迁移

将旧版 VS Code 原生存储迁移到共享文件系统（`~/.cline/data/`），实现跨平台状态共享。

迁移内容包括：API 密钥、自定义指令、任务历史、MCP 目录。

## 阶段 4：核心初始化（common.ts: initialize）

[common.ts](../src/common.ts)

```typescript
async function initialize(storageContext: StorageContext) {
    // 1. 配置 API 端点（读取 endpoints.json）
    configureClineEndpoint()

    // 2. 初始化 StateManager（加载所有状态到内存缓存）
    await StateManager.initialize(storageContext)

    // 3. 初始化错误服务、遥测
    ErrorService.initialize()
    PostHogClientProvider.initialize()

    // 4. 创建 WebviewProvider（通过 HostProvider 工厂）
    const webviewProvider = HostProvider.get().createWebviewProvider()
    webviewProvider.init()  // 注册 gRPC 服务处理函数

    // 5. 启动后台任务
    startSyncWorker()         // 状态同步
    startTempFileCleanup()    // 临时文件清理
    startFileContextCleanup() // 文件上下文清理

    // 6. 遥测
    telemetryService.captureExtensionActivated()
}
```

### StateManager 初始化

[StateManager.ts](../src/core/storage/StateManager.ts)

初始化时从磁盘读取所有状态到内存缓存：

```
StateManager.initialize()
    ├── globalStateCache     ← ~/.cline/data/ 全局设置
    ├── taskStateCache       ← 任务相关状态
    ├── secretsCache         ← API 密钥等
    ├── workspaceStateCache  ← 工作区特定状态
    └── remoteConfigCache    ← 远程配置
```

## 阶段 5：VS Code 命令注册

[extension.ts](../src/extension.ts)

注册所有 VS Code 命令和 UI 元素：

| 类别 | 示例 |
|------|------|
| 侧边栏按钮 | PlusButton（新建任务）、SettingsButton、HistoryButton |
| 代码操作 | Add to Cline、Explain Code、Improve Code、Fix Code |
| Jupyter | Notebook 单元格操作 |
| URI Handler | OAuth 回调（cline://protocol/...） |
| Diff Provider | 自定义差异视图 |
| Webview | 注册侧边栏 Webview（SidebarProvider） |

## 公共 API

[exports/index.ts](../src/exports/index.ts)

`activate()` 返回一个 `ClineAPI` 对象，供其他扩展调用：

```typescript
interface ClineAPI {
    startNewTask(task?: string, images?: string[]): void
    sendMessage(message: string, images?: string[]): void
    pressPrimaryButton(): void
    pressSecondaryButton(): void
}
```

## 关键类关系

```
HostProvider (单例，平台工厂)
    │
    └── creates → WebviewProvider
                      │
                      └── creates → Controller (每个 Webview 实例一个)
                                        │
                                        ├── holds → StateManager (单例)
                                        ├── holds → McpHub
                                        ├── holds → AuthService
                                        └── manages → Task (0或1个活跃)
```
