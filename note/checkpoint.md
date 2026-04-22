# Checkpoint 机制

> 基于 Shadow Git 的对话状态快照与回滚系统，允许用户在对话任意阶段回滚文件和/或对话历史。

## 核心架构：Shadow Git

系统为每个工作区创建一个**独立的 Shadow Git 仓库**，其 `core.worktree` 指向用户的工作目录，在不影响用户自身 Git 仓库的情况下对文件做快照。

```
存储路径: globalStorage/checkpoints/{cwdHash}/.git/
          cwdHash = 工作区路径的 13 位数字哈希

Shadow Git 配置:
  core.worktree     = 用户实际工作区目录
  user.name         = "Cline Checkpoint"
  user.email        = "checkpoint@cline.bot"
  commit.gpgSign    = false
```

### 关键文件

| 文件 | 职责 |
|------|------|
| `src/integrations/checkpoints/CheckpointTracker.ts` | 核心 Git 操作：init、commit、reset |
| `src/integrations/checkpoints/CheckpointGitOperations.ts` | 底层 git 命令封装 |
| `src/integrations/checkpoints/CheckpointExclusions.ts` | 文件排除规则 |
| `src/integrations/checkpoints/CheckpointLockUtils.ts` | 跨实例并发锁 |
| `src/integrations/checkpoints/CheckpointUtils.ts` | 路径哈希、工作区校验 |
| `src/integrations/checkpoints/index.ts` | `TaskCheckpointManager`，上层管理器 |
| `src/integrations/checkpoints/factory.ts` | 工厂函数 `buildCheckpointManager()` |

## Checkpoint 创建

### 创建时机（`src/core/task/index.ts`）

| 时机 | 代码位置 | 说明 |
|------|---------|------|
| 首次 API 请求 | ~L2420-2483 | 任务开始时拍初始快照，发出 `checkpoint_created` 消息 |
| 每轮工具执行完毕 | ~L3180 | `checkpointManager.saveCheckpoint()` |
| 用户发送反馈/消息 | ~L1279 | 处理用户输入前保存 |
| 任务完成 | `index.ts:189-224` | hash 绑定到 `completion_result` 而非新建消息 |

### Commit 流程

```
CheckpointTracker.commit():
  1. tryAcquireCheckpointLockWithRetry()     ← 防并发
  2. renameNestedGitRepos()                  ← .git → .git_disabled
  3. git add . --ignore-errors               ← 暂存文件（排除大文件等）
  4. git commit --allow-empty --no-verify    ← 生成快照
  5. renameNestedGitRepos()                  ← .git_disabled → .git
  6. 释放锁
  7. 发送 CHECKPOINT_COMMIT 事件
```

## Checkpoint 回滚

### 三种回滚类型

| 类型 | 工作区文件 | 对话历史 |
|------|-----------|---------|
| `"task"` | 不变 | 截断到 checkpoint 位置 |
| `"workspace"` | `git reset --hard` 回滚 | 不变 |
| `"taskAndWorkspace"` | `git reset --hard` 回滚 | 截断到 checkpoint 位置 |

### Task 回滚细节（`index.ts:659-703`）

- 对话历史切片到 `message.conversationHistoryIndex + 2`
- `ContextManager.truncateContextHistory()` 截断上下文
- 被删除消息的 API 费用汇总后以 `deleted_api_reqs` 发出
- 对于 `task`-only 回滚，检测 checkpoint 后编辑的文件并作为待处理警告

### Workspace 回滚

`CheckpointTracker.resetHead()` → `git reset --hard <commitHash>` 在 Shadow Git 上执行，由于 `core.worktree` 指向用户目录，实际文件被回滚。

## Diff 查看

`checkpointDiff` handler → `checkpointTracker.getDiffSet(hash)` → `HostProvider.diff.openMultiFileDiff()` 打开 VS Code 多文件 diff 编辑器。

## 文件排除规则（`CheckpointExclusions.ts`）

| 类别 | 示例 |
|------|------|
| Git 目录 | `.git/`, `.git_disabled/` |
| 构建产物 | `node_modules/`, `dist/`, `build/`, `.next/` |
| 媒体文件 | 图片、视频、音频格式 |
| 缓存/临时 | `*.lock`, `*.tmp`, `*.log`, `*.cache` |
| 敏感配置 | `*.env*`, `*.local` |
| 大文件 | 归档 (`*.zip`, `*.tar`), 二进制 (`*.exe`, `*.dll`), 数据库 (`*.sqlite`) |

## Proto 定义

`proto/cline/checkpoints.proto`:

```protobuf
service CheckpointsService {
  rpc checkpointDiff(Int64Request) returns (Empty);
  rpc checkpointRestore(CheckpointRestoreRequest) returns (Empty);
  rpc subscribeToCheckpoints(CheckpointSubscriptionRequest) returns (stream CheckpointEvent);
  rpc getCwdHash(StringArrayRequest) returns (PathHashMap);
}
```

`CheckpointEvent` 流式推送：`CHECKPOINT_INIT`、`CHECKPOINT_COMMIT`、`CHECKPOINT_RESTORE`

## 调用链路

```
UI (CheckmarkControl / CheckpointMenu)
  → gRPC (CheckpointsServiceClient)
    → Controller (checkpointRestore / checkpointDiff)
      → TaskCheckpointManager
        → CheckpointTracker
          → Shadow Git 操作
```

## 并发控制

- 使用文件夹锁 `~/.cline/data/checkpoints/{cwdHash}` 防止多实例并发操作
- VS Code 中跳过锁（编辑器保证单实例）
- 嵌套 `.git` 目录在 staging 时临时重命名以规避 git submodule 要求

## 迁移

`CheckpointMigration.ts` 负责清理旧版按任务存储的 checkpoint 目录（`globalStorage/tasks/{taskId}/checkpoints/`），迁移到新的按工作区集中存储。
