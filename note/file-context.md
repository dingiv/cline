# 文件上下文收集机制

> Cline 如何自动收集相关文件并填充到 prompt 中。核心思路：**无 RAG/Embedding**，采用工具驱动的按需发现 + 环境信息自动注入。

## 整体流程

```
用户发起任务
    │
    ▼
startTask()  → 包装用户文本为 <task>，附加图片/文件
    │
    ▼
initiateTaskLoop()  → includeFileDetails = true（仅首次）
    │
    ▼
recursivelyMakeClineRequests()
    ├── loadContext()
    │     ├── parseMentions()       → 展开 @/path 为 <file_content>
    │     ├── getEnvironmentDetails(true)  → 首次包含完整文件树
    │     └── parseSlashCommands()
    │
    ├── attemptApiRequest()
    │     ├── 加载 .clinerules / .cursorrules / AGENTS.md 等
    │     ├── 获取打开/可见的编辑器标签页
    │     └── getSystemPrompt(context)  → 组装系统提示词
    │
    └── api.createMessage()  → 发送给 AI
          │
          ▼
      AI 使用工具主动探索文件
        ├── read_file          → 读取文件内容
        ├── search_files       → ripgrep 正则搜索
        ├── list_files         → 目录列表
        └── list_code_definition_names  → tree-sitter AST 解析
```

---

## 一、系统提示词中的静态上下文

### System Prompt 模板结构

`src/core/prompts/system-prompt/components/` 定义了 12 个可组合的 Section：

```
AGENT_ROLE → TOOL_USE → TASK_PROGRESS → MCP → EDITING_FILES
→ ACT_VS_PLAN → CAPABILITIES → SKILLS → FEEDBACK → RULES
→ SYSTEM_INFO → OBJECTIVE → USER_INSTRUCTIONS
```

与文件上下文相关的 Section：

| Section | 作用 |
|---------|------|
| `SYSTEM_INFO` | 操作系统、IDE、工作目录、多根工作区路径 |
| `CAPABILITIES` | 告诉模型："首次请求时会提供工作目录的递归文件列表" |
| `RULES` | 指导模型如何分析文件结构、使用文件工具 |
| `USER_INSTRUCTIONS` | 注入 .clinerules、.cursorrules 等规则文件内容 |

### 规则文件自动发现与注入

`src/core/context/instructions/user-instructions/` 中按优先级加载：

| 来源 | 文件 | 说明 |
|------|------|------|
| 全局 `.clinerules` | `cline-rules.ts:21` | `~/.clinerules/` 目录 |
| 本地 `.clinerules` | `cline-rules.ts:82` | `<cwd>/.clinerules/` |
| `.cursorrules` | `external-rules.ts:90` | 工作区根目录 |
| `.windsurfrules` | `external-rules.ts:64` | 工作区根目录 |
| `AGENTS.md` | `external-rules.ts:161` | 递归搜索（需顶层存在） |

规则文件支持 YAML frontmatter 条件过滤（如 `paths:` 匹配当前编辑的文件），通过 `RuleContextBuilder` 评估。

---

## 二、Environment Details（每轮动态上下文）

`src/core/task/index.ts` 的 `getEnvironmentDetails()` 方法（~L3528），每次 API 调用前组装 `<environment_details>` 块：

### 首次请求（`includeFileDetails = true`）

```
<environment_details>
  # VS Code Visible Files        ← 当前可见的编辑器标签页
  # VS Code Open Tabs            ← 所有打开的标签页
  # Actively Running Terminals   ← 终端输出
  # Recently Modified Files      ← FileContextTracker 追踪的外部修改
  # Current Time                 ← 时间和时区

  <file_details>                 ← 仅首次：完整文件树（BFS 遍历，上限 200 项）
    src/
      core/
        task/index.ts
        prompts/...
      ...
  </file_details>

  # Context Window Usage         ← Token 使用百分比
</environment_details>
```

**文件树生成**：`src/services/glob/list-files.ts` 使用 globby BFS 遍历，递归读取 `.gitignore` 避免遍历过大目录，上限 200 项。

### 后续请求（`includeFileDetails = false`）

不再包含完整文件树，但仍包含：可见文件、打开标签页、终端输出、最近修改文件、Token 用量。

---

## 三、工具驱动的文件发现（AI 主动探索）

AI 收到环境信息后，通过 4 个文件工具自主探索代码库：

### read_file

| 项目 | 说明 |
|------|------|
| 定义 | `src/core/prompts/system-prompt/tools/read_file.ts` |
| 处理器 | `src/core/task/tools/handlers/ReadFileToolHandler.ts` |
| 功能 | 读取文件内容，单次上限 1000 行 |
| 去重 | 读取后自动调用 `fileContextTracker.trackFileContext(path, "read_tool")` |
| 防冗余 | 同一文件读取 3+ 次后插入 `[DUPLICATE READ]` 警告 |

### search_files

| 项目 | 说明 |
|------|------|
| 定义 | `src/core/prompts/system-prompt/tools/search_files.ts` |
| 处理器 | `src/core/task/tools/handlers/SearchFilesToolHandler.ts` |
| 引擎 | `src/services/ripgrep/index.ts` — 封装 `rg` 命令，JSON 输出解析 |
| 限制 | 最多 300 条结果，输出上限 0.25MB，匹配行前后各 1 行上下文 |

### list_files

| 项目 | 说明 |
|------|------|
| 定义 | `src/core/prompts/system-prompt/tools/list_files.ts` |
| 处理器 | `src/core/task/tools/handlers/ListFilesToolHandler.ts` |
| 功能 | 列出目录内容（递归或顶层），上限 200 项 |

### list_code_definition_names

| 项目 | 说明 |
|------|------|
| 定义 | `src/core/prompts/system-prompt/tools/list_code_definition_names.ts` |
| 处理器 | `src/core/task/tools/handlers/ListCodeDefinitionNamesToolHandler.ts` |
| 引擎 | `src/services/tree-sitter/index.ts` — WebAssembly tree-sitter 解析 |
| 语言 | JS/TS/Python/Rust/Go/C/C++/C#/Ruby/Java/PHP/Swift/Kotlin |
| 功能 | 提取顶层定义名称（类、函数、变量等），单次上限 50 文件 |

---

## 四、@提及系统（用户触发的上下文注入）

`src/core/mentions/index.ts` 的 `parseMentions()` 解析用户输入中的 `@` 语法：

| 语法 | 效果 |
|------|------|
| `@/path/to/file` | 读取文件内容，内联为 `<file_content path="...">` |
| `@/path/to/folder/` | 列出目录并读取所有非二进制文件 |
| `@workspace:path` | 多根工作区指定路径 |
| `@url` | 通过 headless browser 抓取 URL 内容 |
| `@problems` | 获取工作区诊断信息 |
| `@git-changes` | 获取 git working state |
| `@<hash>` | 获取指定 commit 信息 |

所有提及的文件自动注册到 `FileContextTracker` 进行追踪。

---

## 五、FileContextTracker（文件追踪与过时检测）

`src/core/context/context-tracking/FileContextTracker.ts`

### 追踪的操作类型

```
"read_tool"      → read_file 工具读取
"user_edited"    → 用户外部编辑
"cline_edited"   → Cline 自身编辑
"file_mentioned" → @提及或文件附件
```

### 过时检测机制

1. 每个被追踪的文件注册一个 `chokidar` watcher
2. 当用户在外部编辑器修改了已追踪的文件 → 加入 `recentlyModifiedFiles`
3. 下一轮 `getEnvironmentDetails()` 中以 `# Recently Modified Files` 警告注入
4. 如果任务暂停期间有文件被修改，额外插入 `<explicit_instructions>` 强制重新读取

---

## 六、上下文窗口管理

`src/core/context/context-management/ContextManager.ts`

当 token 用量接近模型上下文窗口限制时，依次执行：

### 策略 1：文件读取去重（L796-942）

扫描对话历史中的重复文件读取（`read_file`、`replace_in_file`、`write_to_file`、`@file`），将旧副本替换为：
```
[[NOTE] This file read has been removed to save space. The most recent version is available in a later message.]
```

### 策略 2：对话截断（L299-339）

移除较早的 user-assistant 消息对（移除一半或四分之一），保留第一对。

### 策略 3：上下文压缩（摘要）

`src/core/prompts/contextManagement.ts` 的 `summarizeTask()` 指示模型生成详细摘要，包含 "Required Files" 列出关键文件路径。

---

## 七、用户文件附件

`src/integrations/misc/extract-text.ts` 的 `processFilesIntoText()`（L199）：

用户拖放文件到聊天时，读取文件内容包装为 `<file_content>` 标签注入。支持纯文本、PDF、DOCX、Jupyter Notebook、Excel，400KB 截断。

---

## 关键设计总结

| 特性 | 实现方式 |
|------|---------|
| RAG/Embedding | **无** — 不使用向量搜索或语义检索 |
| 首次上下文 | 完整文件树（200 项上限）+ 规则文件 + 打开的标签页 |
| 后续上下文 | 可见文件 + 打开标签页 + 最近修改文件 |
| 文件发现 | ripgrep 正则搜索 + tree-sitter AST 解析 |
| 文件读取 | 按需读取，自动追踪，重复读取警告 |
| 过时检测 | chokidar watcher 监听外部修改 |
| 上下文压缩 | 去重 → 截断 → 摘要 |
