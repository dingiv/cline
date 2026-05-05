# 工具执行

## 核心文件

- [ToolExecutor.ts](/src/core/task/ToolExecutor.ts) — 工具执行入口
- [ToolExecutorCoordinator.ts](/src/core/task/tools/ToolExecutorCoordinator.ts) — Handler 注册与路由
- [ToolValidator.ts](/src/core/task/tools/ToolValidator.ts) — 参数验证
- [autoApprove.ts](/src/core/task/tools/autoApprove.ts) — 自动审批逻辑
- [handlers/](/src/core/task/tools/handlers/) — 各工具的具体实现

## 执行流程

```
presentAssistantMessage()
    │ 解析出 tool_use content block
    ▼
ToolExecutor.executeTool(block)
    │
    ├── 1. ToolValidator.validate(block)  ← 参数校验
    ├── 2. autoApprove.check(tool, params) ← 检查是否需要用户确认
    │       ├── 需要确认 → say("tool") → 等待用户 approve/reject
    │       └── 自动通过 → 直接执行
    ├── 3. 运行 pre-tool-use hooks
    ├── 4. ToolExecutorCoordinator.execute(config, block)
    │       └── handlers.get(block.name).execute(config, block)
    ├── 5. 运行 post-tool-use hooks
    └── 6. 返回 ToolResponse（字符串）→ 加入 userMessageContent
```

## 26 个工具处理器

| 工具名                       | Handler 文件                                                                              | 功能                |
| ---------------------------- | ----------------------------------------------------------------------------------------- | ------------------- |
| `read_file`                  | [ReadFileToolHandler](../src/core/task/tools/handlers/)                                   | 读取文件内容        |
| `write_to_file`              | [WriteToFileToolHandler](../src/core/task/tools/handlers/)                                | 创建或完整覆盖文件  |
| `replace_in_file`            | [WriteToFileToolHandler](../src/core/task/tools/handlers/)                                | 搜索替换文件内容    |
| `apply_patch`                | [ApplyPatchHandler](../src/core/task/tools/handlers/)                                     | 应用 diff 补丁      |
| `execute_command`            | [ExecuteCommandToolHandler](../src/core/task/tools/handlers/ExecuteCommandToolHandler.ts) | 执行 shell 命令     |
| `search_files`               | [SearchFilesToolHandler](../src/core/task/tools/handlers/)                                | 正则搜索文件内容    |
| `list_files`                 | [ListFilesToolHandler](../src/core/task/tools/handlers/)                                  | 列出目录内容        |
| `list_code_definition_names` | [ListCodeDefinitionNamesToolHandler](../src/core/task/tools/handlers/)                    | 搜索代码符号        |
| `browser_action`             | [BrowserToolHandler](../src/core/task/tools/handlers/)                                    | 浏览器自动化操作    |
| `web_fetch`                  | [WebFetchToolHandler](../src/core/task/tools/handlers/)                                   | 获取网页内容        |
| `web_search`                 | [WebSearchToolHandler](../src/core/task/tools/handlers/)                                  | 网页搜索            |
| `use_mcp_tool`               | [UseMcpToolHandler](../src/core/task/tools/handlers/)                                     | 调用 MCP 服务器工具 |
| `access_mcp_resource`        | [AccessMcpResourceHandler](../src/core/task/tools/handlers/)                              | 访问 MCP 资源       |
| `load_mcp_documentation`     | [LoadMcpDocumentationHandler](../src/core/task/tools/handlers/)                           | 加载 MCP 文档       |
| `ask_followup_question`      | [AskFollowupQuestionToolHandler](../src/core/task/tools/handlers/)                        | 向用户提问          |
| `attempt_completion`         | [AttemptCompletionHandler](../src/core/task/tools/handlers/)                              | 标记任务完成        |
| `new_task`                   | [NewTaskHandler](../src/core/task/tools/handlers/)                                        | 创建子任务          |
| `plan_mode_respond`          | [PlanModeRespondHandler](../src/core/task/tools/handlers/)                                | 计划模式回复        |
| `act_mode_respond`           | [ActModeRespondHandler](../src/core/task/tools/handlers/)                                 | 行动模式回复        |
| `condense`                   | [CondenseHandler](../src/core/task/tools/handlers/)                                       | 压缩上下文          |
| `summarize_task`             | [SummarizeTaskHandler](../src/core/task/tools/handlers/)                                  | 总结任务            |
| `report_bug`                 | [ReportBugHandler](../src/core/task/tools/handlers/)                                      | 报告 bug            |
| `new_rule`                   | [WriteToFileToolHandler](../src/core/task/tools/handlers/)                                | 创建规则文件        |
| `generate_explanation`       | [GenerateExplanationHandler](../src/core/task/tools/handlers/)                            | 生成变更解释        |
| `use_skill`                  | [UseSkillToolHandler](../src/core/task/tools/handlers/)                                   | 执行技能            |
| `use_subagents`              | [UseSubagentsToolHandler](../src/core/task/tools/handlers/)                               | 子代理调度          |

## 工具调用格式

### XML 格式（大多数模型）

```xml
<read_file>
<path>src/index.ts</path>
</read_file>
```

### 原生 JSON 格式（支持原生工具调用的模型）

```json
{
    "name": "read_file",
    "arguments": {
        "path": "src/index.ts"
    }
}
```

解析逻辑在 [assistant-message/index.ts](../src/core/assistant-message/index.ts)，根据 `enableNativeToolCalls` 标志选择解析方式。

## 自动审批机制

[autoApprove.ts](../src/core/task/tools/autoApprove.ts)

```typescript
interface AutoApprovalSettings {
    enabled: boolean
    actions: {
        readFiles: boolean         // 读取文件
        writeFiles: boolean        // 写入文件
        executeSafeCommands: boolean // 安全命令（ls, cat, etc.）
        executeAllCommands: boolean  // 所有命令
        browserActions: boolean     // 浏览器操作
        mcpTools: boolean          // MCP 工具
    }
    maxTrustedCalls: number        // 最大连续自动审批次数
}
```

审批流程：
1. 检查工具类型是否在自动审批列表中
2. 检查连续自动审批计数是否超限
3. 如果需要审批 → 发送 `ask("tool")` 到 UI，等待用户响应
4. 用户可以：批准、拒绝、始终批准（更新设置）

## Hook 系统

[hooks/](../src/core/hooks/)

在工具执行前后运行的钩子：

```
Pre-tool-use Hook:
    ├── 可以取消工具执行
    ├── 可以注入额外的上下文信息
    └── 可以修改工具参数

Post-tool-use Hook:
    ├── 可以处理工具输出
    └── 可以触发后续操作
```

Hook 配置存储在 `.cline/hooks.json` 中。

## 子代理（Subagents）

[subagent/](../src/core/task/tools/subagent/)

`use_subagents` 工具支持将子任务分派给独立的代理实例：

- 每个子代理有独立的上下文和工具集
- 子代理结果汇总后返回给主代理
- 支持 `new_task` 工具创建独立的子任务

## 命令执行安全

[ExecuteCommandToolHandler.ts](../src/core/task/tools/handlers/ExecuteCommandToolHandler.ts)

```typescript
// 命令执行流程
execute_command(command)
    ├── 1. clineIgnoreController 检查（是否忽略该路径）
    ├── 2. commandPermissionController 检查（命令是否允许）
    ├── 3. 用户审批（如需）
    ├── 4. commandExecutor.execute(command, cwd)
    │       ├── 在 VS Code 终端中执行
    │       ├── 实时输出流式回传
    │       └── 支持超时和中止
    └── 5. 返回命令输出（stdout + stderr）
```
