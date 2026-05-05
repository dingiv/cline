# 项目结构

## 顶层目录

```
cline/
├── src/                  # 扩展核心源码（TypeScript）
├── webview-ui/src/       # Webview 前端（React SPA）
├── cli/                  # 命令行版本（React Ink）
├── proto/cline/          # Protobuf 接口定义（16个.proto文件）
├── standalone/           # 独立桌面版
│
├── tests/                # 集成测试
├── evals/                # 评测脚本
├── scripts/              # 构建/代码生成脚本
├── locales/              # 国际化翻译文件
├── assets/               # 图标等静态资源
├── walkthrough/          # VS Code 引导教程
├── dist/                 # 编译输出
└── package.json          # 扩展清单（name: "claude-dev"）
```

## src/ 核心目录

```
src/
├── extension.ts          # VS Code 入口（activate/deactivate）
├── common.ts             # 平台无关的初始化逻辑
├── exports/              # 公共 API（供其他扩展调用）
│   ├── index.ts          # ClineAPI: startNewTask, sendMessage, pressButton
│   └── cline.d.ts        # 类型声明
│
├── core/                 # ★ 核心业务逻辑
│   ├── api/              #   AI 提供商（47个）
│   ├── assistant-message/ #   助手消息解析
│   ├── commands/         #   VS Code 命令注册
│   ├── context/          #   上下文管理
│   ├── controller/       #   控制器（中央调度）
│   ├── hooks/            #   钩子系统（工具执行前后）
│   ├── ignore/           #   .clineignore 支持
│   ├── locks/            #   并发锁
│   ├── mentions/         #   @提及处理
│   ├── permissions/      #   权限管理
│   ├── prompts/          #   ★ 系统提示词构建
│   ├── slash-commands/   #   斜杠命令
│   ├── storage/          #   ★ 状态持久化
│   ├── task/             #   ★ 任务/Agent 循环
│   ├── webview/          #   Webview 提供者基类
│   └── workspace/        #   工作区管理
│
├── generated/            # 自动生成的 gRPC 代码
│   ├── grpc-js/          #   服务端实现
│   ├── nice-grpc/        #   Promise 客户端
│   └── hosts/            #   Host 桥接处理
│
├── hosts/                # 平台适配层
│   ├── host-provider.ts  #   依赖注入容器（单例）
│   └── vscode/           #   VS Code 特定实现
│
├── integrations/         # 第三方集成
│   ├── terminal/         #   终端管理
│   ├── editor/           #   编辑器集成
│   └── diagnostics/      #   诊断信息
│
├── services/             # 通用服务
├── shared/               # ★ 共享代码（前后端共用）
│   ├── api.ts            #   提供商/模型类型定义
│   ├── ExtensionMessage.ts # 扩展→Webview 消息类型
│   ├── WebviewMessage.ts   # Webview→扩展 消息类型
│   ├── proto/            #   生成的 Proto 类型
│   ├── proto-conversions/ #   Proto ↔ 业务类型转换
│   ├── storage/          #   状态键定义
│   └── providers/        #   providers.json（提供商列表）
│
├── utils/                # 工具函数
├── packages/             # 子包
└── __tests__/            # 单元测试 + 快照
```

## Webview UI 目录

```
webview-ui/src/
├── main.tsx              # React 入口
├── App.tsx               # 根组件，视图路由
├── Providers.tsx         # Context Provider 嵌套
├── components/
│   ├── chat/             # ★ 聊天界面
│   │   ├── ChatView.tsx         # 主聊天容器
│   │   ├── ChatRow.tsx          # 单条消息渲染（核心，处理各种类型）
│   │   ├── ChatTextArea.tsx     # 输入框
│   │   ├── chat-view/           # 模块化子组件/hooks/utils
│   │   ├── task-header/         # 任务头、上下文窗口、焦点链
│   │   └── auto-approve-menu/   # 自动审批栏
│   ├── settings/         # 设置面板（8个标签页）
│   ├── history/          # 历史任务浏览
│   ├── mcp/              # MCP 服务器管理
│   ├── account/          # 账户/额度
│   └── onboarding/       # 首次运行引导
├── services/
│   ├── grpc-client-base.ts  # ProtoBus 客户端基类
│   └── grpc-client.ts       # ★ 自动生成的 13 个 Service Client
├── context/              # React Context
│   ├── ExtensionStateContext.tsx  # 主状态（~935行）
│   └── ClineAuthContext.tsx       # 认证状态
└── utils/                # 工具函数
```

## Proto 定义

```
proto/cline/
├── common.proto          # 通用类型（Empty, StringRequest, Int64Request）
├── task.proto            # 任务操作（创建、取消、历史）
├── ui.proto              # UI 事件（按钮、部分消息、ClineSay 枚举）
├── state.proto           # 状态订阅、设置更新
├── models.proto          # 模型信息、ApiProvider 枚举
├── account.proto         # 认证（登录/登出、额度）
├── browser.proto         # 浏览器连接
├── checkpoints.proto     # 检查点操作
├── commands.proto        # 上下文菜单命令
├── file.proto            # 文件操作
├── hooks.proto           # 钩子管理
├── mcp.proto             # MCP 服务器 CRUD
├── slash.proto           # 斜杠命令
├── web.proto             # Web 操作
├── worktree.proto        # Git Worktree
└── oca_account.proto     # OCA 认证
```
