# Prompt 构建

## 核心文件

- [system-prompt/index.ts](../src/core/prompts/system-prompt/index.ts) — 入口，`getSystemPrompt()` 函数
- [registry/](../src/core/prompts/system-prompt/registry/) — PromptRegistry + PromptBuilder
- [variants/](../src/core/prompts/system-prompt/variants/) — 13 个模型变体
- [components/](../src/core/prompts/system-prompt/components/) — 12 个可复用组件
- [tools/](../src/core/prompts/system-prompt/tools/) — 23 个工具定义

## 架构设计

```
getSystemPrompt(promptContext)
    │
    ├── PromptRegistry 选择匹配的 variant（基于模型家族）
    │
    ├── PromptBuilder 组装最终提示词
    │     ├── 加载 variant 的模板/布局
    │     ├── 对于每个 SECTION：
    │     │     ├── variant 有 override？→ 使用 variant 的版本
    │     │     └── 否则 → 使用共享 component
    │     └── 解析 {{PLACEHOLDER}} 模板变量
    │
    └── 返回 { systemPrompt: string, nativeTools?: ClineTool[] }
```

## 变体系统（Variants）

每个模型家族有自己的变体配置：

| 变体 | 目录 | 适用模型 |
|------|------|----------|
| generic | [generic/](../src/core/prompts/system-prompt/variants/generic/) | 默认回退，所有未匹配的模型 |
| next-gen | [next-gen/](../src/core/prompts/system-prompt/variants/next-gen/) | Claude 4 系列 |
| native-next-gen | [native-next-gen/](../src/core/prompts/system-prompt/variants/native-next-gen/) | Claude 4（原生工具调用） |
| gpt-5 | [gpt-5/](../src/core/prompts/system-prompt/variants/gpt-5/) | GPT-5 |
| native-gpt-5 | [native-gpt-5/](../src/core/prompts/system-prompt/variants/native-gpt-5/) | GPT-5（原生工具调用） |
| native-gpt-5-1 | [native-gpt-5-1/](../src/core/prompts/system-prompt/variants/native-gpt-5-1/) | GPT-5.1 |
| gemini-3 | [gemini-3/](../src/core/prompts/system-prompt/variants/gemini-3/) | Gemini 2.5 |
| glm | [glm/](../src/core/prompts/system-prompt/variants/glm/) | GLM 系列 |
| hermes | [hermes/](../src/core/prompts/system-prompt/variants/hermes/) | Hermes 模型 |
| devstral | [devstral/](../src/core/prompts/system-prompt/variants/devstral/) | Devstral |
| trinity | [trinity/](../src/core/prompts/system-prompt/variants/trinity/) | Trinity |
| xs | [xs/](../src/core/prompts/system-prompt/variants/xs/) | 小型/本地模型（高度精简） |

### 变体匹配机制

```typescript
// variants/*/config.ts
export const config: VariantConfig = {
    matcher: (modelId: string, provider: string) => boolean,  // 匹配函数
    sections: [SECTION.RULES, SECTION.CAPABILITIES, ...],     // 启用的 SECTION
    componentOverrides: {                                      // 组件覆盖
        [SECTION.RULES]: customRulesTemplate,
    },
    tools: [ClineDefaultTool.READ_FILE, ...],                  // 启用的工具
}
```

### 变体优先级

PromptRegistry 按注册顺序匹配，第一个返回 true 的变体生效。如果都不匹配，回退到 `generic`。

### 三种覆盖方式

1. **componentOverrides**：替换某个组件的内容
   ```typescript
   componentOverrides: {
       [SECTION.RULES]: myCustomRulesTemplate
   }
   ```

2. **自定义 template**：完全自定义布局
   ```typescript
   // variants/next-gen/template.ts
   export const rules_template = `自定义规则模板 {{OBJECTIVE}} ...`
   ```

3. **什么都不做**：使用共享 component 的默认内容

## 组件系统（Components）

12 个可复用的提示词组件，每个对应一个 SECTION：

| 组件 | SECTION | 内容 |
|------|---------|------|
| `agent_role` | ROLE | AI 代理的角色定义 |
| `system_info` | SYSTEM_INFO | 系统信息（OS、Shell、工作目录等） |
| `rules` | RULES | 行为规则和约束 |
| `objective` | OBJECTIVE | 任务目标和格式要求 |
| `capabilities` | CAPABILITIES | 能力声明 |
| `tool_use` | TOOL_USE | 工具使用说明（XML 格式） |
| `editing_files` | EDITING_FILES | 文件编辑规则 |
| `mcp` | MCP | MCP 服务器工具说明 |
| `user_instructions` | USER_INSTRUCTIONS | 用户自定义指令（.clinerules） |
| `skills` | SKILLS | 可用技能列表 |
| `act_vs_plan_mode` | ACT_VS_PLAN_MODE | 行动/计划模式说明 |
| `feedback` | FEEDBACK | 反馈相关说明 |

## 工具定义（Tools）

[tools/](../src/core/prompts/system-prompt/tools/) 下每个文件定义一个工具：

```
tools/
├── init.ts                # 注册所有工具变体
├── read_file.ts           # 读取文件
├── write_to_file.ts       # 写入文件
├── replace_in_file.ts     # 替换文件内容
├── apply_patch.ts         # 应用补丁
├── execute_command.ts     # 执行命令
├── search_files.ts        # 搜索文件
├── list_files.ts          # 列出文件
├── list_code_definition_names.ts  # 列出代码定义
├── browser_action.ts      # 浏览器操作
├── web_fetch.ts           # 获取网页
├── web_search.ts          # 网页搜索
├── use_mcp_tool.ts        # 使用 MCP 工具
├── access_mcp_resource.ts # 访问 MCP 资源
├── ask_followup_question.ts  # 追问用户
├── attempt_completion.ts  # 尝试完成任务
├── new_task.ts            # 创建子任务
├── plan_mode_respond.ts   # 计划模式响应
├── act_mode_respond.ts    # 行动模式响应
├── condense.ts            # 压缩上下文
├── generate_explanation.ts # 生成解释
├── use_skill.ts           # 使用技能
└── ...                    # 更多工具
```

### 工具变体

每个工具可以定义不同模型家族的变体：

```typescript
// tools/read_file.ts
export const read_file_variants = [
    GENERIC_VARIANT,           // XML 格式，所有模型通用
    NATIVE_NEXT_GEN_VARIANT,   // JSON schema，Claude 4 原生工具
    XS_VARIANT,                // 精简版，适合小模型
]
```

**回退机制**：如果某个模型家族没有特定变体，`ClineToolSet.getToolByNameWithFallback()` 自动回退到 GENERIC。

## SystemPromptContext

构建系统提示词时传入的上下文对象：

```typescript
interface SystemPromptContext {
    cwd: string                              // 当前工作目录
    ide: string                              // IDE 类型
    provider: string                         // API 提供商
    modelId: string                          // 模型 ID
    modelFamily: ModelFamily                 // 模型家族
    mcpHub?: McpHub                          // MCP 服务器中心
    skills?: Skill[]                         // 可用技能
    globalRules?: Rule[]                     // 全局规则
    localRules?: Rule[]                      // 项目规则
    externalRules?: Rule[]                   # 外部规则
    browserSettings: BrowserSettings         // 浏览器设置
    mode: "plan" | "act"                     // 当前模式
    workspaceRoots: string[]                 // 工作区根目录
    enableNativeToolCalls?: boolean          // 是否启用原生工具调用
    // ... 更多字段
}
```

## XS 变体特殊处理

[xs/](../src/core/prompts/system-prompt/variants/xs/) 为小模型做了大量精简：
- 工具描述更短
- 规则更简洁
- 模板直接内联在 [template.ts](../src/core/prompts/system-prompt/variants/xs/template.ts) 中（不使用组件系统）
- 减少上下文占用

## 测试与快照

```bash
# 修改 prompt 后必须更新快照
UPDATE_SNAPSHOTS=true npm run test:unit
```

快照位于 [__snapshots__/](../src/__tests__/__snapshots__/)，验证各模型家族在不同上下文下的提示词输出。
