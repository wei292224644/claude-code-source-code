# Layer 08: BACKGROUND TASKS — 后台任务

## 概述

**BACKGROUND TASKS** 是异步守护进程操作。模型可以在后台运行慢任务，不阻塞主对话。位于 `src/tasks/`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 任务框架 | `src/utils/task/framework.ts` |
| 任务类型 | `src/tasks/types.ts` |
| DreamTask | `src/tasks/DreamTask/DreamTask.ts` |
| LocalShellTask | `src/tasks/LocalShellTask/` |
| 任务停止 | `src/tasks/stopTask.ts` |
| 任务输出 | `src/utils/task/diskOutput.ts` |

---

## 任务类型

```typescript
// src/Task.ts:6-13
export type TaskType =
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'local_workflow'
  | 'monitor_mcp'
  | 'dream'
```

| 类型 | 描述 |
|------|------|
| `dream` | 后台思考/研究任务 |
| `local_bash` | 本地 Shell 后台执行 |
| `local_agent` | 本地代理任务 |
| `remote_agent` | 远程代理任务 |
| `in_process_teammate` | 进程内队友任务 |
| `local_workflow` | 本地工作流 |
| `monitor_mcp` | MCP 监控任务 |

---

## 任务状态

```typescript
// src/Task.ts:15-20
export type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

---

## DreamTask - 后台思考任务

### DreamTaskState

```typescript
// src/tasks/DreamTask/DreamTask.ts:25-41
export type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  abortController?: AbortController
  priorMtime: number
}

export type DreamPhase = 'starting' | 'updating'
```

### 注册 DreamTask

```typescript
// src/tasks/DreamTask/DreamTask.ts:52-74
export function registerDreamTask(
  setAppState: SetAppState,
  opts: {
    sessionsReviewing: number
    priorMtime: number
    abortController: AbortController
  },
): string {
  const id = generateTaskId('dream')
  const task: DreamTaskState = {
    ...createTaskStateBase(id, 'dream', 'dreaming'),
    type: 'dream',
    status: 'running',
    phase: 'starting',
    sessionsReviewing: opts.sessionsReviewing,
    filesTouched: [],
    turns: [],
    abortController: opts.abortController,
    priorMtime: opts.priorMtime,
  }
  registerTask(task, setAppState)
  return id
}
```

### 添加 DreamTurn

```typescript
// src/tasks/DreamTask/DreamTask.ts:76-104
export function addDreamTurn(
  taskId: string,
  turn: DreamTurn,
  touchedPaths: string[],
  setAppState: SetAppState,
): void {
  updateTaskState<DreamTaskState>(taskId, setAppState, task => {
    const seen = new Set(task.filesTouched)
    const newTouched = touchedPaths.filter(p => !seen.has(p) && seen.add(p))
    // 避免空更新的重新渲染
    if (turn.text === '' && turn.toolUseCount === 0 && newTouched.length === 0) {
      return task
    }
    return {
      ...task,
      phase: newTouched.length > 0 ? 'updating' : task.phase,
      filesTouched: [...task.filesTouched, ...newTouched],
      turns: task.turns.slice(-(MAX_TURNS - 1)).concat(turn),
    }
  })
}
```

### 完成 DreamTask

```typescript
// src/tasks/DreamTask/DreamTask.ts:106-120
export function completeDreamTask(
  taskId: string,
  setAppState: SetAppState,
): void {
  updateTaskState<DreamTaskState>(taskId, setAppState, task => ({
    ...task,
    status: 'completed',
    endTime: Date.now(),
    notified: true,
    abortController: undefined,
  }))
}
```

---

## Task 框架

### 注册任务

```typescript
// src/utils/task/framework.ts:77-117
export function registerTask(task: TaskState, setAppState: SetAppState): void {
  let isReplacement = false
  setAppState(prev => {
    const existing = prev.tasks[task.id]
    isReplacement = existing !== undefined
    // 合并已有状态（用于 resume）
    const merged = existing && 'retain' in existing
      ? { ...task, retain: existing.retain, startTime: existing.startTime, ... }
      : task
    return { ...prev, tasks: { ...prev.tasks, [task.id]: merged } }
  })

  if (isReplacement) return  // resume 不是新开始

  enqueueSdkEvent({ type: 'system', subtype: 'task_started', ... })
}
```

### 更新任务状态

```typescript
// src/utils/task/framework.ts:48-72
export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId] as T | undefined
    if (!task) return prev
    const updated = updater(task)
    if (updated === task) return prev  // 早期返回，避免无变化时的重新渲染
    return { ...prev, tasks: { ...prev.tasks, [taskId]: updated } }
  })
}
```

### 驱逐终端任务

```typescript
// src/utils/task/framework.ts:125-144
export function evictTerminalTask(
  taskId: string,
  setAppState: SetAppState,
): void {
  setAppState(prev => {
    const task = prev.tasks?.[taskId]
    if (!task) return prev
    if (!isTerminalTaskStatus(task.status)) return prev
    if (!task.notified) return prev
    // 检查保留期
    if ('retain' in task && (task.evictAfter ?? Infinity) > Date.now()) return prev
    const { [taskId]: _, ...remainingTasks } = prev.tasks
    return { ...prev, tasks: remainingTasks }
  })
}
```

---

## 任务输出

### 轮询间隔

```typescript
// src/utils/task/framework.ts:22
export const POLL_INTERVAL_MS = 1000
```

### 任务附件

```typescript
// src/utils/task/framework.ts:31-39
export type TaskAttachment = {
  type: 'task_status'
  taskId: string
  toolUseId?: string
  taskType: TaskType
  status: TaskStatus
  description: string
  deltaSummary: string | null
}
```

---

## 后台任务工作流程

```
主循环中调用慢操作
    ↓
创建后台任务
    ↓
registerTask(task, setAppState)
    ↓
立即返回 (不阻塞)
    ↓
后台继续执行
    ↓
添加 turn: addDreamTurn()
    ↓
完成时: completeDreamTask()
    ↓
注入通知到主对话
```

---

## DreamTask 详细流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DreamTask 生命周期                              │
│                                                                     │
│  1. registerDreamTask()                                           │
│     - 生成任务 ID                                                  │
│     - 创建 DreamTaskState                                          │
│     - 注册到 AppState                                              │
│     - 设置 status: 'running'                                       │
│                                                                     │
│  2. 后台执行思考/研究                                              │
│     - agent 在后台运行                                             │
│     - 调用 addDreamTurn() 更新进度                                  │
│                                                                     │
│  3. addDreamTurn()                                                │
│     - 更新 filesTouched                                            │
│     - 添加 turns (最多保留 MAX_TURNS=30)                          │
│     - phase: 'starting' → 'updating'                             │
│                                                                     │
│  4. completeDreamTask()                                           │
│     - status: 'completed'                                          │
│     - endTime: Date.now()                                         │
│     - notified: true                                              │
│                                                                     │
│  5. 驱逐 (evictTerminalTask)                                      │
│     - 等待 notified                                               │
│     - 从 AppState 移除                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 任务与 TodoWrite 的区别

| 特性 | 任务 | TodoWrite |
|------|------|----------|
| 执行位置 | **后台** | 主循环内 |
| 状态持久化 | **磁盘** | 仅内存 |
| 长期运行 | **支持** | 阻塞主循环 |
| 通知 | **有** | 无 |
| UI 显示 | **Footer pill, Shift+Down** | 无 |

---

## 关键设计

### 1. 状态合并

```typescript
// registerTask 中对 resume 的处理
const merged = existing && 'retain' in existing
  ? {
      ...task,
      retain: existing.retain,           // 保留 UI 状态
      startTime: existing.startTime,      // 保持排序稳定
      messages: existing.messages,         // 保留已查看的记录
    }
  : task
```

### 2. 无变化时早期返回

```typescript
// updateTaskState 中避免无意义的重新渲染
if (updated === task) return prev
```

### 3. 终端状态检查

```typescript
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### 4. 轮询清理

```typescript
// 轮询间隔
export const POLL_INTERVAL_MS = 1000

// 被终止任务的显示时间
export const STOPPED_DISPLAY_MS = 3_000
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| DreamTask | `src/tasks/DreamTask/DreamTask.ts` |
| registerDreamTask | `src/tasks/DreamTask/DreamTask.ts:52-74` |
| addDreamTurn | `src/tasks/DreamTask/DreamTask.ts:76-104` |
| completeDreamTask | `src/tasks/DreamTask/DreamTask.ts:106-120` |
| registerTask | `src/utils/task/framework.ts:77-117` |
| updateTaskState | `src/utils/task/framework.ts:48-72` |
| evictTerminalTask | `src/utils/task/framework.ts:125-144` |
| TaskType | `src/Task.ts:6-13` |
| TaskStatus | `src/Task.ts:15-20` |
| POLL_INTERVAL_MS | `src/utils/task/framework.ts:22` |

---

*基于 Claude Code v2.1.88 源代码分析*
