# Layer 07: PERSISTENT TASKS — 持久化任务

## 概述

**PERSISTENT TASKS** 是基于文件的任务系统，任务状态持久化到磁盘，不受对话结束影响。位于 `src/tools/TaskCreateTool/`、`src/utils/tasks.ts`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 任务工具 | `src/tools/TaskCreateTool/` |
| 任务更新 | `src/tools/TaskUpdateTool/` |
| 任务获取 | `src/tools/TaskGetTool/` |
| 任务列表 | `src/tools/TaskListTool/` |
| 任务停止 | `src/tools/TaskStopTool/` |
| 任务输出 | `src/tools/TaskOutputTool/` |
| 任务核心实现 | `src/utils/tasks.ts` |
| 任务类型定义 | `src/Task.ts` |

---

## 四个核心工具

| 工具 | 描述 | 状态位置 |
|------|------|----------|
| `TaskCreateTool` | 创建任务 | `~/.claude/tasks/<id>.json` |
| `TaskUpdateTool` | 更新任务状态 | 文件系统 |
| `TaskGetTool` | 获取任务详情 | 持久化 |
| `TaskListTool` | 列出所有任务 | - |

---

## 任务状态

### TaskState 定义

```typescript
// src/utils/tasks.ts:69-89
export const TASK_STATUSES = ['pending', 'in_progress', 'completed'] as const

export const TaskSchema = lazySchema(() =>
  z.object({
    id: z.string(),
    subject: z.string(),
    description: z.string(),
    activeForm: z.string().optional(),
    owner: z.string().optional(),
    status: TaskStatusSchema(),
    blocks: z.array(z.string()),
    blockedBy: z.array(z.string()),
    metadata: z.record(z.string(), z.unknown()).optional(),
  }),
)
```

### TaskState 完整定义

```typescript
// src/Task.ts:45-57
export type TaskStateBase = {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  toolUseId?: string
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string
  outputOffset: number
  notified: boolean
}
```

### TaskStatus 枚举

| 状态 | 描述 |
|------|------|
| `pending` | 等待执行 |
| `in_progress` | 执行中 |
| `completed` | 已完成 |
| `failed` | 失败 |
| `killed` | 被终止 |

---

## 任务存储结构

```
~/.claude/
└── tasks/
    ├── .highwatermark         # 最高任务 ID 记录
    ├── 1.json                 # 任务 1
    ├── 2.json                 # 任务 2
    └── <taskId>.json          # 每个任务一个 JSON 文件
```

### 任务文件格式

```json
// ~/.claude/tasks/1.json
{
  "id": "1",
  "subject": "实现用户认证",
  "description": "使用 JWT 实现用户认证功能",
  "activeForm": "实现用户认证",
  "owner": "agent-123",
  "status": "in_progress",
  "blocks": [],
  "blockedBy": [],
  "metadata": {
    "priority": "high"
  }
}
```

---

## createTask - 创建任务

```typescript
// src/utils/tasks.ts:284-308
export async function createTask(
  taskListId: string,
  taskData: Omit<Task, 'id'>,
): Promise<string> {
  const lockPath = await ensureTaskListLockFile(taskListId)

  let release: (() => Promise<void>) | undefined
  try {
    // 获取文件锁
    release = await lockfile.lock(lockPath, LOCK_OPTIONS)

    // 读取最高 ID
    const highestId = await findHighestTaskId(taskListId)
    const id = String(highestId + 1)

    // 创建任务文件
    const task: Task = { id, ...taskData }
    const path = getTaskPath(taskListId, id)
    await writeFile(path, jsonStringify(task, null, 2))

    notifyTasksUpdated()  // 通知 UI 更新
    return id
  } finally {
    if (release) {
      await release()
    }
  }
}
```

### 文件锁机制

```typescript
// src/utils/tasks.ts:102-108
// 防止并发创建任务时的竞争条件
const LOCK_OPTIONS = {
  retries: {
    retries: 30,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

---

## updateTask - 更新任务

```typescript
// src/utils/tasks.ts:370-391
export async function updateTask(
  taskListId: string,
  taskId: string,
  updates: Partial<Omit<Task, 'id'>>,
): Promise<Task | null> {
  // 先检查任务是否存在
  const taskBeforeLock = await getTask(taskListId, taskId)
  if (!taskBeforeLock) {
    return null
  }

  let release: (() => Promise<void>) | undefined
  try {
    release = await lockfile.lock(path, LOCK_OPTIONS)
    return await updateTaskUnsafe(taskListId, taskId, updates)
  } finally {
    await release?.()
  }
}

// 无锁版本，供内部使用
async function updateTaskUnsafe(
  taskListId: string,
  taskId: string,
  updates: Partial<Omit<Task, 'id'>>,
): Promise<Task | null> {
  const existing = await getTask(taskListId, taskId)
  if (!existing) {
    return null
  }
  const updated: Task = { ...existing, ...updates, id: taskId }
  const path = getTaskPath(taskListId, taskId)
  await writeFile(path, jsonStringify(updated, null, 2))
  notifyTasksUpdated()
  return updated
}
```

---

## TaskCreateTool

```typescript
// src/tools/TaskCreateTool/TaskCreateTool.ts:48-138
export const TaskCreateTool = buildTool({
  name: TASK_CREATE_TOOL_NAME,
  shouldDefer: true,
  isEnabled() {
    return isTodoV2Enabled()
  },

  async call({ subject, description, activeForm, metadata }, context) {
    const taskId = await createTask(getTaskListId(), {
      subject,
      description,
      activeForm,
      status: 'pending',
      owner: undefined,
      blocks: [],
      blockedBy: [],
      metadata,
    })

    // 执行任务创建钩子
    const blockingErrors: string[] = []
    const generator = executeTaskCreatedHooks(
      taskId, subject, description, getAgentName(), getTeamName(),
      undefined, context?.abortController?.signal, undefined, context,
    )
    for await (const result of generator) {
      if (result.blockingError) {
        blockingErrors.push(getTaskCreatedHookMessage(result.blockingError))
      }
    }

    if (blockingErrors.length > 0) {
      await deleteTask(getTaskListId(), taskId)
      throw new Error(blockingErrors.join('\n'))
    }

    // 创建任务时自动展开任务列表
    context.setAppState(prev => {
      if (prev.expandedView === 'tasks') return prev
      return { ...prev, expandedView: 'tasks' as const }
    })

    return {
      data: { task: { id: taskId, subject } },
    }
  },
})
```

---

## TaskListId 解析

```typescript
// src/utils/tasks.ts:199-215
export function getTaskListId(): string {
  // 优先级：
  // 1. CLAUDE_CODE_TASK_LIST_ID - 显式任务列表 ID
  // 2. 进程内队友：队长的团队名称（队友共享队长的任务列表）
  // 3. CLAUDE_CODE_TEAM_NAME - 进程队友
  // 4. 队长团队名称
  // 5. 会话 ID - 独立会话的后备
}
```

---

## 任务依赖管理

### blocks 和 blockedBy

```typescript
// src/utils/tasks.ts:420-435
// 删除任务时，清理引用它的任务
const allTasks = await listTasks(taskListId)
for (const task of allTasks) {
  const newBlocks = task.blocks.filter(id => id !== taskId)
  const newBlockedBy = task.blockedBy.filter(id => id !== taskId)
  if (newBlocks.length !== task.blocks.length ||
      newBlockedBy.length !== task.blockedBy.length) {
    await updateTask(taskListId, task.id, {
      blocks: newBlocks,
      blockedBy: newBlockedBy,
    })
  }
}
```

---

## TodoWriteTool vs TaskTool

| 特性 | TodoWrite | Task |
|------|------------|------|
| 持久化 | 仅内存 | **磁盘** |
| 跨会话 | **否** | **是** |
| 状态机 | 简单复选框 | 完整状态 |
| 优先级 | 无 | 支持 metadata |
| 依赖 | 无 | blocks/blockedBy |

---

## 任务创建流程

```
TaskCreateTool(subject="实现用户认证")
    ↓
createTask(getTaskListId(), { subject, status: 'pending' })
    ↓
获取文件锁
    ↓
findHighestTaskId() 读取最高 ID
    ↓
id = highestId + 1
    ↓
写入 ~/.claude/tasks/<id>.json
    ↓
notifyTasksUpdated() 通知 UI
    ↓
释放文件锁
    ↓
返回 taskId
```

---

## 关键设计

### 1. 文件锁机制

防止多个 Claude 实例并发创建任务时的竞争条件。

### 2. 高水位标记

```typescript
// 防止任务 ID 复用
const HIGH_WATER_MARK_FILE = '.highwatermark'
// 删除任务后，最高 ID 仍被保留
// 新任务使用更高的 ID
```

### 3. 任务通知

```typescript
// 任务更新时通知 UI 刷新
export function notifyTasksUpdated(): void {
  try {
    tasksUpdated.emit()
  } catch {
    // 忽略通知失败，任务变更必须成功
  }
}
```

### 4. 状态迁移

```typescript
// 旧状态名称迁移
if (data.status === 'open') data.status = 'pending'
else if (data.status === 'resolved') data.status = 'completed'
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| createTask | `src/utils/tasks.ts:284-308` |
| updateTask | `src/utils/tasks.ts:370-391` |
| getTask | `src/utils/tasks.ts:310-350` |
| deleteTask | `src/utils/tasks.ts:393-450` |
| getTaskListId | `src/utils/tasks.ts:199-215` |
| TaskCreateTool | `src/tools/TaskCreateTool/TaskCreateTool.ts:48-138` |
| TaskSchema | `src/utils/tasks.ts:76-89` |
| TASK_STATUSES | `src/utils/tasks.ts:69` |
| LOCK_OPTIONS | `src/utils/tasks.ts:102-108 |

---

*基于 Claude Code v2.1.88 源代码分析*
