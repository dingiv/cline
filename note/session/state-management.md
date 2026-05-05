# 状态管理

## 核心文件

- [StateManager.ts](../src/core/storage/StateManager.ts) — 中央状态管理器（单例）
- [disk.ts](../src/core/storage/disk.ts) — 文件持久化
- [state-migrations.ts](../src/core/storage/state-migrations.ts) — 旧版迁移
- [state-keys.ts](../src/shared/storage/state-keys.ts) — 状态键类型定义
- [storage-context.ts](../src/shared/storage/storage-context.ts) — 存储上下文抽象

## StateManager 设计

```
┌─────────────────────────────────────────────────┐
│                  StateManager (单例)              │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐               │
│  │ globalState  │  │ taskState    │               │
│  │ Cache        │  │ Cache        │               │
│  └──────┬──────┘  └──────┬───────┘               │
│         │                │                        │
│         │  ┌─────────────┴──────────┐             │
│         │  │ secretsCache           │             │
│         │  │ workspaceStateCache    │             │
│         │  │ sessionOverrideCache   │             │
│         │  │ remoteConfigCache      │             │
│         └──┴────────────────────────┘             │
│                                                   │
│         │  ┌─────────────────────────┐            │
│         └─►│ Debounced Persistence   │            │
│            │ (500ms 批量写磁盘)       │            │
│            └─────────────────────────┘            │
└───────────────────────────────────────────────────┘
```

### 读写策略

```typescript
// 读：立即从内存缓存返回（无 IO）
getGlobalState<T>(key: string): T

// 写：立即更新内存 + 延迟写磁盘
setGlobalState(key: string, value: unknown): void {
    globalStateCache[key] = value          // 立即更新内存
    pendingGlobalState.add(key)            // 标记脏
    schedulePersistence(500ms)             // 延迟批量写入
}
```

### 脏标记追踪

```typescript
pendingGlobalState: Set<string>      // 需要持久化的全局状态键
pendingTaskState: Set<string>        // 需要持久化的任务状态键
pendingSecrets: Set<string>          // 需要持久化的密钥
pendingWorkspaceState: Set<string>   // 需要持久化的工作区状态键
```

500ms 内的所有写入会批量合并为一次磁盘操作。

## 存储后端

### StorageContext 抽象

```typescript
interface StorageContext {
    globalState: KeyValueStore     // 全局设置
    workspaceState: KeyValueStore  // 工作区特定状态
    secrets: SecretStore           // API 密钥等敏感数据
}
```

### 文件存储位置

```
~/.cline/
├── data/                           # 全局数据
│   ├── settings.json               # 全局设置
│   ├── taskHistory.json            # 任务历史索引
│   ├── cline_mcp_settings.json     # MCP 服务器配置
│   ├── openrouter_models.json      # 缓存的模型列表
│   ├── mcp_marketplace_catalog.json
│   └── remote_config_{orgId}.json
│
└── tasks/                          # 任务数据
    └── {taskId}/
        ├── api_conversation_history.json  # 原始 API 对话
        ├── ui_messages.json              # UI 显示的消息
        ├── context_history.json          # 上下文管理历史
        └── task_metadata.json            # 任务元数据
```

### 原子写入

文件写入使用 temp + rename 模式保证原子性：

```typescript
// disk.ts
async function writeAtomic(filePath: string, content: string) {
    const tempPath = filePath + ".tmp"
    await fs.writeFile(tempPath, content)
    await fs.rename(tempPath, filePath)  // 原子操作
}
```

## 多实例行为

```
VS Code 窗口 A                    VS Code 窗口 B
┌──────────────────┐              ┌──────────────────┐
│ StateManager #1  │              │ StateManager #2  │
│ ├─ 独立内存缓存   │              │ ├─ 独立内存缓存   │
│ ├─ 独立脏标记     │              │ ├─ 独立脏标记     │
│ └─ 写入同一磁盘   │──► 文件 ◄───│ └─ 写入同一磁盘   │
└──────────────────┘              └──────────────────┘

问题：窗口 B 的缓存不会自动感知窗口 A 的磁盘写入
解决：file watcher 监听 taskHistory.json 变化 → 触发同步
```

## 状态键定义

[state-keys.ts](../src/shared/storage/state-keys.ts)

```typescript
interface GlobalState {
    // API 配置
    apiProvider: ApiProvider
    apiModelId: string
    apiKey: string
    // ... 更多设置

    // 任务历史
    taskHistory: HistoryItem[]

    // 自动审批
    autoApprovalSettings: AutoApprovalSettings

    // 浏览器设置
    browserSettings: BrowserSettings

    // MCP
    mcpServers: McpServer[]

    // ... 约 100 个字段
}
```

## Webview 端状态管理

[ExtensionStateContext.tsx](../webview-ui/src/context/ExtensionStateContext.tsx)

使用 React Context + useState，没有外部状态库：

```typescript
// 状态水合流程
1. 组件挂载
2. 调用 StateServiceClient.subscribeToState()（流式）
3. 后端推送 stateJson
4. 解析 JSON → setState() 合并到 ExtensionState
5. 所有子组件通过 useExtensionState() 获取状态
```

## 跨窗口状态读取

```
场景：窗口 A 设置了状态 → 立即打开窗口 B

问题：窗口 B 的 StateManager 缓存尚未从磁盘加载

解决：在启动阶段（initialize）直接读取 context.globalState.get()
而非使用 StateManager 的缓存

// src/common.ts
const value = context.globalState.get<string>("myKey")  // 直读，不经过缓存
```

## 添加新状态键的完整步骤

| 步骤 | 文件 | 操作 |
|------|------|------|
| 1 | [state-keys.ts](../src/shared/storage/state-keys.ts) | 定义类型 |
| 2 | [state-helpers.ts](../src/core/storage/utils/state-helpers.ts) | readGlobalStateFromDisk() 中添加读取 |
| 3 | StateManager | setGlobalState/getGlobalStateKey |
| 4a | [updateSettings.ts](../src/core/controller/state/updateSettings.ts) | webview 更新路径 |
| 4b | [updateSettingsCli.ts](../src/core/controller/state/updateSettingsCli.ts) | CLI 更新路径 |
| 4c | [state.proto](../proto/cline/state.proto) | UpdateSettingsRequest |
| 4d | — | npm run protos 重新生成 |
| 4e | [controller/index.ts](../src/core/controller/index.ts) | getStateToPostToWebview() |
| 4f | [ExtensionMessage.ts](../src/shared/ExtensionMessage.ts) | 类型+默认值 |
