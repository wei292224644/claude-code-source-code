# Tasks/Subagent 模块技术文档

## 概述

本文档描述 Claude Code 中 **Tasks/Subagent** 模块的完整设计与实现。该模块负责管理子代理（Subagent）的生命周期，支持三种执行模式：本地代理（LocalAgent）、远程代理（RemoteAgent）和进程内队友（InProcessTeammate）。

**源码位置**：`src/tasks/`

---

## 1. Subagent 设计理念

### 1.1 为什么需要 Subagent

Subagent（子代理）系统解决以下核心问题：

| 需求 | 说明 |
|------|------|
| **隔离上下文** | 子代理拥有独立的消息历史和文件状态缓存，避免污染主对话 |
| **并行处理** | 多个子代理可以同时执行不同任务，提高吞吐量 |
| **专用能力** | 可以为子代理配置特定模型、工具集或权限模式 |
| **资源解耦** | 子代理在独立任务状态中运行，崩溃不会影响主流程 |

### 1.2 与直接函数调用的区别

```
直接函数调用:
  主循环 → 调用函数 → 等待返回 → 继续
                    ↓
              共享上下文/状态

Subagent:
  主循环 → 创建子代理任务 → 调度执行 → 异步收集结果
                    ↓
              独立上下文/状态
```

**关键区别**：
- **状态隔离**：Subagent 有独立的 `readFileState` 克隆
- **消息独立**：子代理的消息历史不混入主对话
- **执行解耦**：子代理通过 `AbortController` 独立中止
- **结果聚合**：父代理显式收集子代理结果

### 1.3 三种模式的适用场景

| 模式 | TaskType | 执行位置 | 适用场景 |
|------|----------|----------|----------|
| **LocalAgentTask** | `local_agent` | 同进程，后台运行 | 长期运行的后台任务、需要文件隔离的工作 |
| **RemoteAgentTask** | `remote_agent` | 云端 Claude.ai | 超长上下文、需要更强算力的任务 |
| **InProcessTeammateTask** | `in_process_teammate` | 同进程，协作模式 | 实时协作、快速响应、团队工作流 |

---

## 2. Task 类型定义

### 2.1 TaskType 枚举

```typescript
// src/Task.ts:6-13
export type TaskType =
  | 'local_bash'           // 本地 Bash 任务
  | 'local_agent'          // 本地代理任务
  | 'remote_agent'         // 远程代理任务
  | 'in_process_teammate'  // 进程内队友
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监控
  | 'dream'                // 梦幻任务
```

### 2.2 TaskStatus 状态机

```typescript
// src/Task.ts:15-20
export type TaskStatus =
  | 'pending'    // 任务已创建，等待执行
  | 'running'    // 任务执行中
  | 'completed'  // 任务成功完成
  | 'failed'     // 任务执行失败
  | 'killed'     // 任务被强制终止

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### 2.3 TaskStateBase 基础状态

所有任务类型都继承此基类：

```typescript
// src/Task.ts:45-57
export type TaskStateBase = {
  id: string              // 唯一任务 ID，前缀标识类型：a=local_agent, r=remote_agent, t=in_process_teammate
  type: TaskType           // 任务类型
  status: TaskStatus       // 当前状态
  description: string       // 任务描述（用于 UI 显示）
  toolUseId?: string       // 关联的 tool_use ID
  startTime: number         // 开始时间戳
  endTime?: number         // 结束时间戳
  totalPausedMs?: number   // 暂停总时长
  outputFile: string        // 输出文件路径
  outputOffset: number      // 输出文件读取偏移
  notified: boolean         // 是否已发送通知
}
```

### 2.4 Task 基类接口

```typescript
// src/Task.ts:72-76
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

### 2.5 TaskResult 类型（结果聚合）

子代理执行结果通过 `AgentToolResult` 传递：

```typescript
export type AgentToolResult = {
  agentId: string
  result: string           // 执行结果文本
  success: boolean
  error?: string
  toolUses?: number        // 消耗的 tool_use 次数
  totalTokens?: number     // 消耗的总 token 数
  durationMs?: number      // 执行时长
}
```

### 2.6 TaskProgress 类型（进度上报）

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:33-39
export type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]
  summary?: string
}

export type ToolActivity = {
  toolName: string
  input: Record<string, unknown>
  activityDescription?: string
  isSearch?: boolean
  isRead?: boolean
}
```

---

## 3. Task 基类实现

### 3.1 任务 ID 生成规则

```typescript
// src/Task.ts:98-106
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

export function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type] ?? 'x'
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

### 3.2 任务状态基类工厂函数

```typescript
// src/Task.ts:108-125
export function createTaskStateBase(
  id: string,
  type: TaskType,
  description: string,
  toolUseId?: string,
): TaskStateBase {
  return {
    id,
    type,
    status: 'pending',
    description,
    toolUseId,
    startTime: Date.now(),
    outputFile: getTaskOutputPath(id),
    outputOffset: 0,
    notified: false,
  }
}
```

### 3.3 任务注册框架

```typescript
// src/utils/task/framework.ts:77-100
export function registerTask(task: TaskState, setAppState: SetAppState): void {
  let isReplacement = false
  setAppState(prev => {
    const existing = prev.tasks[task.id]
    isReplacement = existing !== undefined
    // 重新注册时保留 UI 状态
    const merged = isReplacement && existing
      ? {
          ...task,
          retain: existing.retain,
          messages: existing.messages,
          diskLoaded: existing.diskLoaded,
          evictAfter: existing.evictAfter,
        }
      : task
    return {
      ...prev,
      tasks: { ...prev.tasks, [task.id]: merged },
    }
  })
}

export function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T
): void {
  setAppState(prev => {
    const task = prev.tasks[taskId]
    if (!task) return prev
    return {
      ...prev,
      tasks: { ...prev.tasks, [taskId]: updater(task as T) },
    }
  })
}
```

---

## 4. LocalAgentTask（本地代理任务）

### 4.1 概述

`LocalAgentTask` 用于在本地进程中创建和管理后台代理任务。代理在独立的异步上下文中运行，结果通过 `AgentToolResult` 返回。

**源码**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx`

### 4.2 LocalAgentTaskState 状态类型

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:116-148
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  unregisterCleanup?: () => void
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  retrieved: boolean
  messages?: Message[]
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean
  pendingMessages: string[]
  retain: boolean
  diskLoaded: boolean
  evictAfter?: number
}

export function isLocalAgentTask(task: unknown): task is LocalAgentTaskState {
  return typeof task === 'object' && task !== null && 'type' in task && task.type === 'local_agent'
}
```

### 4.3 Task 对象定义

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:270-276
export const LocalAgentTask: Task = {
  name: 'LocalAgentTask',
  type: 'local_agent',
  async kill(taskId, setAppState) {
    killAsyncAgent(taskId, setAppState)
  }
}
```

### 4.4 注册异步代理

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:466-515
export function registerAsyncAgent({
  agentId,
  description,
  prompt,
  selectedAgent,
  setAppState,
  parentAbortController,
  toolUseId
}: {
  agentId: string
  description: string
  prompt: string
  selectedAgent: AgentDefinition
  setAppState: SetAppState
  parentAbortController?: AbortController
  toolUseId?: string
}): LocalAgentTaskState {
  void initTaskOutputAsSymlink(agentId, getAgentTranscriptPath(asAgentId(agentId)))

  // 创建中止控制器——如果提供了父级，则创建自动随父级中止的子控制器
  const abortController = parentAbortController
    ? createChildAbortController(parentAbortController)
    : createAbortController()

  const taskState: LocalAgentTaskState = {
    ...createTaskStateBase(agentId, 'local_agent', description, toolUseId),
    type: 'local_agent',
    status: 'running',
    agentId,
    prompt,
    selectedAgent,
    agentType: selectedAgent.agentType ?? 'general-purpose',
    abortController,
    retrieved: false,
    lastReportedToolCount: 0,
    lastReportedTokenCount: 0,
    isBackgrounded: true,
    pendingMessages: [],
    retain: false,
    diskLoaded: false
  }

  // 注册清理处理器
  const unregisterCleanup = registerCleanup(async () => {
    killAsyncAgent(agentId, setAppState)
  })
  taskState.unregisterCleanup = unregisterCleanup

  // 在 AppState 中注册任务
  registerTask(taskState, setAppState)
  return taskState
}
```

### 4.5 完整执行流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LocalAgentTask 执行流程                           │
│                                                                     │
│  1. registerAsyncAgent({ agentId, prompt, selectedAgent })          │
│     ├── 生成任务 ID（如 "a1b2c3d4"）                                │
│     ├── 创建 AbortController（可选：父级关联）                       │
│     ├── 初始化 TaskState（status='running'）                        │
│     ├── 注册清理处理器                                              │
│     └── registerTask(taskState, setAppState)                       │
│                                                                     │
│  2. 代理执行（独立异步循环）                                         │
│     ├── runAgent() → query() 循环                                   │
│     ├── 独立的 readFileState 克隆                                    │
│     ├── 独立的 abortController                                       │
│     └── 结果写入 outputFile                                         │
│                                                                     │
│  3. 进度追踪                                                        │
│     ├── updateProgressFromMessage()                                 │
│     ├── 周期性摘要（后台服务）                                        │
│     └── emitTaskProgress() → SDK 消费者                             │
│                                                                     │
│  4. 完成/失败                                                       │
│     ├── completeAgentTask(result, setAppState)                      │
│     ├── failAgentTask(error, setAppState)                           │
│     └── evictTaskOutput()                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. RemoteAgentTask（远程代理任务）

### 5.1 概述

`RemoteAgentTask` 用于创建和管理运行在 Claude.ai 云端的远程代理任务。通过 HTTP 协议与远程会话通信，支持轮询获取结果。

**源码**：`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

### 5.2 RemoteAgentTaskState 状态类型

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:22-59
export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: RemoteTaskType
  remoteTaskMetadata?: RemoteTaskMetadata
  sessionId: string
  command: string
  title: string
  todoList: TodoList
  log: SDKMessage[]
  isLongRunning?: boolean
  pollStartedAt: number
  isRemoteReview?: boolean
  reviewProgress?: {
    stage?: 'finding' | 'verifying' | 'synthesizing'
    bugsFound: number
    bugsVerified: number
    bugsRefuted: number
  }
  isUltraplan?: boolean
  ultraplanPhase?: Exclude<UltraplanPhase, 'running'>
}

export type RemoteTaskType =
  | 'remote-agent'
  | 'ultraplan'
  | 'ultrareview'
  | 'autofix-pr'
  | 'background-pr'
```

### 5.3 远程任务完成检查器

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:77-86
export type RemoteTaskCompletionChecker = (
  remoteTaskMetadata: RemoteTaskMetadata | undefined
) => Promise<string | null>

const completionCheckers = new Map<RemoteTaskType, RemoteTaskCompletionChecker>()

export function registerCompletionChecker(
  remoteTaskType: RemoteTaskType,
  checker: RemoteTaskCompletionChecker
): void {
  completionCheckers.set(remoteTaskType, checker)
}
```

### 5.4 预条件检查

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:124-141
export async function checkRemoteAgentEligibility({
  skipBundle = false
}: { skipBundle?: boolean } = {}): Promise<RemoteAgentPreconditionResult> {
  const errors = await checkBackgroundRemoteSessionEligibility({ skipBundle })
  if (errors.length > 0) {
    return { eligible: false, errors }
  }
  return { eligible: true }
}
```

### 5.5 完整执行流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RemoteAgentTask 执行流程                          │
│                                                                     │
│  1. 创建远程会话                                                    │
│     ├── checkRemoteAgentEligibility() → 预条件检查                   │
│     ├── fetchSession() → 创建远程会话                               │
│     └── registerTask() → AppState 注册                              │
│                                                                     │
│  2. 轮询获取结果                                                    │
│     ├── pollRemoteSessionEvents() → 事件流                          │
│     ├── 提取 <remote-review> 等标签                                 │
│     └── 检查 completionCheckers                                     │
│                                                                     │
│  3. 消息通知                                                        │
│     ├── enqueueRemoteNotification() → 任务完成通知                   │
│     ├── emitTaskTerminatedSdk() → SDK 事件                          │
│     └── 写入 outputFile                                             │
│                                                                     │
│  4. 清理                                                            │
│     ├── archiveRemoteSession() → 归档会话                           │
│     └── removeRemoteAgentMetadata() → 删除元数据                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. InProcessTeammateTask（进程内队友任务）

### 6.1 概述

`InProcessTeammateTask` 用于创建进程内队友，支持实时协作。队友共享 AppState，通过 AsyncLocalStorage 实现上下文隔离。

**源码**：
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`
- `src/tasks/InProcessTeammateTask/types.ts`

### 6.2 TeammateIdentity 身份类型

```typescript
// src/tasks/InProcessTeammateTask/types.ts:13-20
export type TeammateIdentity = {
  agentId: string       // 如 "researcher@my-team"
  agentName: string     // 如 "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
}
```

### 6.3 InProcessTeammateTaskState 状态类型

```typescript
// src/tasks/InProcessTeammateTask/types.ts:22-76
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'

  identity: TeammateIdentity

  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController
  currentWorkAbortController?: AbortController

  awaitingPlanApproval: boolean
  permissionMode: PermissionMode

  error?: string
  result?: AgentToolResult
  progress?: AgentProgress

  messages?: Message[]
  inProgressToolUseIDs?: Set<string>
  pendingUserMessages: string[]

  spinnerVerb?: string
  pastTenseVerb?: string

  isIdle: boolean
  shutdownRequested: boolean

  onIdleCallbacks?: Array<() => void>

  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

### 6.4 Task 对象定义

```typescript
// src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:24-30
export const InProcessTeammateTask: Task = {
  name: 'InProcessTeammateTask',
  type: 'in_process_teammate',
  async kill(taskId, setAppState) {
    killInProcessTeammate(taskId, setAppState)
  }
}
```

### 6.5 关键辅助函数

```typescript
// src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx

// 请求关闭队友
export function requestTeammateShutdown(
  taskId: string,
  setAppState: SetAppState
): void {
  updateTaskState<InProcessTeammateTaskState>(taskId, setAppState, task => {
    if (task.status !== 'running' || task.shutdownRequested) return task
    return { ...task, shutdownRequested: true }
  })
}

// 向队友对话历史追加消息
export function appendTeammateMessage(
  taskId: string,
  message: Message,
  setAppState: SetAppState
): void {
  updateTaskState<InProcessTeammateTaskState>(taskId, setAppState, task => {
    if (task.status !== 'running') return task
    return { ...task, messages: appendCappedMessage(task.messages, message) }
  })
}

// 向队友待处理队列注入用户消息
export function injectUserMessageToTeammate(
  taskId: string,
  message: string,
  setAppState: SetAppState
): void {
  updateTaskState<InProcessTeammateTaskState>(taskId, setAppState, task => {
    if (isTerminalTaskStatus(task.status)) {
      logForDebugging(`Dropping message for teammate task ${taskId}...`)
      return task
    }
    return {
      ...task,
      pendingUserMessages: [...task.pendingUserMessages, message],
      messages: appendCappedMessage(
        task.messages,
        createUserMessage({ content: message })
      )
    }
  })
}

// 按 agent ID 查找队友任务
export function findTeammateTaskByAgentId(
  agentId: string,
  tasks: Record<string, TaskStateBase>
): InProcessTeammateTaskState | undefined {
  let fallback: InProcessTeammateTaskState | undefined
  for (const task of Object.values(tasks)) {
    if (isInProcessTeammateTask(task) && task.identity.agentId === agentId) {
      if (task.status === 'running') return task
      if (!fallback) fallback = task
    }
  }
  return fallback
}
```

### 6.6 消息上限保护

```typescript
// src/tasks/InProcessTeammateTask/types.ts:101-121
export const TEAMMATE_MESSAGES_UI_CAP = 50

export function appendCappedMessage<T>(
  prev: readonly T[] | undefined,
  item: T
): T[] {
  if (prev === undefined || prev.length === 0) return [item]
  if (prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1))
    next.push(item)
    return next
  }
  return [...prev, item]
}
```

### 6.7 完整执行流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                 InProcessTeammateTask 执行流程                     │
│                                                                     │
│  1. 创建队友                                                        │
│     ├── 生成确定性 agentId: formatAgentId(name, teamName)           │
│     ├── 创建独立的 AbortController                                  │
│     ├── 创建 TeammateIdentity                                       │
│     ├── 在 AppState 中注册任务                                      │
│     └── 初始化 TeammateContext（AsyncLocalStorage）                 │
│                                                                     │
│  2. 消息通信                                                        │
│     ├── injectUserMessageToTeammate() → 注入用户消息                 │
│     ├── appendTeammateMessage() → 追加对话历史                       │
│     └── pendingUserMessages 队列                                    │
│                                                                     │
│  3. 生命周期管理                                                    │
│     ├── requestTeammateShutdown() → 请求关闭                         │
│     ├── killInProcessTeammate() → 强制终止                           │
│     └── onIdleCallbacks → 空闲通知                                  │
│                                                                     │
│  4. 协作完成                                                        │
│     ├── 队长等待: waitForTeammatesToBecomeIdle()                    │
│     ├── 结果收集 → 邮箱系统                                         │
│     └── 清理 → TeamDeleteTool                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. 结果聚合

### 7.1 父 Agent 如何收集子代理结果

#### 1. 任务通知机制

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:197-262
function enqueueAgentNotification({
  taskId,
  description,
  status,
  error,
  setAppState,
  finalMessage,
  usage,
  toolUseId,
  worktreePath,
  worktreeBranch
}: {
  taskId: string
  description: string
  status: 'completed' | 'failed' | 'killed'
  error?: string
  setAppState: SetAppState
  finalMessage?: string
  usage?: { totalTokens: number; toolUses: number; durationMs: number }
  toolUseId?: string
  worktreePath?: string
  worktreeBranch?: string
}): void {
  // 原子性检查和设置 notified 标志
  let shouldEnqueue = false
  updateTaskState<LocalAgentTaskState>(taskId, setAppState, task => {
    if (task.notified) return task
    shouldEnqueue = true
    return { ...task, notified: true }
  })
  if (!shouldEnqueue) return

  // 构建通知消息（XML 标签格式）
  const message = `<${TASK_NOTIFICATION_TAG}>
<${TASK_ID_TAG}>${taskId}</${TASK_ID_TAG}>
<${STATUS_TAG}>${status}</${STATUS_TAG}>
<${SUMMARY_TAG}>...</${SUMMARY_TAG}>
<result>${finalMessage}</result>
<usage>...</usage>
</${TASK_NOTIFICATION_TAG}>`

  enqueuePendingNotification({ value: message, mode: 'task-notification' })
}
```

#### 2. 并行子代理的 Await 模式

父代理可以同时启动多个子代理并并行等待结果：

```typescript
// 伪代码示例
const subagentPromises = [
  spawn_task('local_agent', { prompt: '任务1' }),
  spawn_task('local_agent', { prompt: '任务2' }),
  spawn_task('local_agent', { prompt: '任务3' }),
]

// 并行等待所有结果
const results = await Promise.all(subagentPromises)
```

### 7.2 结果合并策略

| 策略 | 说明 | 使用场景 |
|------|------|----------|
| **Append** | 多个子代理结果追加到消息历史 | 并行研究、多角度分析 |
| **Replace** | 子代理结果替换当前上下文 | 验证代理覆盖原结果 |
| **Aggregate** | 结构化合并（如 bug 列表） | 代码审查、测试修复 |

---

## 8. 完整数据流图

### 8.1 总体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Claude Code 主进程                              │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         AppState Store                                │  │
│  │  ┌──────────────────────────────────────────────────────────────┐    │  │
│  │  │                   tasks: Record<TaskId, TaskState>            │    │  │
│  │  │  ┌────────────┐ ┌────────────┐ ┌────────────┐             │    │  │
│  │  │  │LocalAgent  │ │LocalAgent  │ │InProcess   │  ...          │    │  │
│  │  │  │TaskState   │ │TaskState   │ │Teammate    │               │    │  │
│  │  │  │(a*****)    │ │(a*****)    │ │TaskState   │               │    │  │
│  │  │  └────────────┘ └────────────┘ │(t*****)    │               │    │  │
│  │  │                                  └────────────┘               │    │  │
│  │  └──────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                   │
│  │LocalAgent   │    │LocalAgent   │    │InProcess    │                   │
│  │Task         │    │Task         │    │Teammate     │                   │
│  │             │    │             │    │Task         │                   │
│  │• abortCtrl  │    │• abortCtrl  │    │• ALS Context│                   │
│  │• Progress   │    │• Progress   │    │• Messages   │                   │
│  │• Result     │    │• Result     │    │• onIdle     │                   │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                   │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                   │
│  │ Background  │    │ Background  │    │ In-Process  │                   │
│  │ AsyncAgent │    │ AsyncAgent │    │ Runner      │                   │
│  │ (runAgent) │    │ (runAgent) │    │             │                   │
│  └─────────────┘    └─────────────┘    └─────────────┘                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP / 轮询
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Claude.ai 云端                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    RemoteAgentTask                                    │  │
│  │  ┌────────────┐ ┌────────────┐                                     │  │
│  │  │ Session 1  │ │ Session 2  │  ...                                 │  │
│  │  │ (ultraplan)│ │ (autofix)  │                                     │  │
│  │  └────────────┘ └────────────┘                                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 子代理创建时序图

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  主循环   │    │ AgentTool │    │ register │    │ AppState │    │ runAgent │
│          │    │          │    │  Task    │    │  Store   │    │  循环    │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │               │
     │ call(prompt)  │               │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │               │ createAgentId │               │               │
     │               │───────────────│               │               │
     │               │               │               │               │
     │               │ registerTask()│               │               │
     │               │───────────────│───────────────>               │
     │               │               │               │               │
     │               │               │  return OK   │               │
     │               │<──────────────│<──────────────│               │
     │               │               │               │               │
     │               │               │               │  start loop  │
     │               │               │               │<─────────────│
     │               │               │               │               │
     │               │               │               │  yield msg  │
     │               │               │               │─────────────>│
     │               │               │               │               │
     │  return stream│               │               │               │
     │<──────────────│               │               │               │
     │               │               │               │               │
```

### 8.3 消息通知流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          任务完成通知流程                                     │
│                                                                             │
│  1. 子代理完成任务                                                           │
│     runAgent() → 正常结束或异常                                              │
│                    │                                                         │
│                    ▼                                                         │
│  2. completeAgentTask() / failAgentTask()                                    │
│     ├── 更新 TaskState.status = 'completed' / 'failed'                      │
│     ├── 设置 TaskState.result / TaskState.error                               │
│     └── 设置 endTime                                                         │
│                    │                                                         │
│                    ▼                                                         │
│  3. enqueueAgentNotification()                                               │
│     ├── 检查 notified 标志（原子操作）                                        │
│     ├── 构建 XML 通知消息                                                    │
│     │   <task-notification>                                                 │
│     │     <task-id>...</task-id>                                           │
│     │     <status>completed</status>                                       │
│     │     <summary>Agent "xxx" completed</summary>                          │
│     │     <result>...</result>                                             │
│     │     <usage><total_tokens>...</total_tokens></usage>                   │
│     │   </task-notification>                                                │
│     └── enqueuePendingNotification()                                        │
│                    │                                                         │
│                    ▼                                                         │
│  4. 消息队列处理                                                              │
│     消息进入待处理队列，等待主循环消费                                         │
│                    │                                                         │
│                    ▼                                                         │
│  5. 主循环处理通知                                                            │
│     ├── 解析 <task-notification> 标签                                        │
│     ├── 注入结果到主对话上下文                                               │
│     └── 模型继续处理                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.4 协作模式数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           团队协作数据流                                      │
│                                                                             │
│  ┌────────────────┐         ┌────────────────┐                              │
│  │   Team Lead    │         │   Teammate 1    │                              │
│  │  (主代理进程)   │         │  (InProcess)    │                              │
│  │                │         │                │                              │
│  │  AppState      │◄───────►│  Teammate      │                              │
│  │  TeamContext   │         │  Context       │                              │
│  │                │         │  (ALS)         │                              │
│  └───────┬────────┘         └───────┬────────┘                              │
│          │                          │                                       │
│          │ injectUserMessage()      │ appendTeammateMessage()              │
│          │                          │                                       │
│          │ pendingUserMessages ─────┼──► 消息队列                          │
│          │                          │                                       │
│          │              ┌────────────────┐                                 │
│          │              │   Mailbox      │                                 │
│          │              │  File System   │                                 │
│          │              │ ~/.claude/     │                                 │
│          │              │ teams/.../     │                                 │
│          │              └────────────────┘                                 │
│          │                          │                                       │
│          │              ┌────────────────┐                                 │
│          │              │   Teammate 2   │                                 │
│          │              │  (InProcess)   │                                 │
│          │              └────────────────┘                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 统一抽象层设计（Python 端口需补充）

三种执行模式（LocalAgent / RemoteAgent / InProcessTeammate）当前各自维护独立的状态结构和结果格式，Python 端口需要建立统一的抽象屏障。

### 9.1 统一结果接口

无论哪种执行模式，最终都应返回一致的 `TaskResult`：

```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class TaskStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    KILLED = "killed"

@dataclass(frozen=True)
class TaskResult:
    """所有任务类型的统一结果格式"""
    task_id: str
    status: TaskStatus
    output: str                          # 文本输出（失败时为空）
    error: Optional[str]                 # 错误信息（成功时为 None）
    success: bool                        # status in (COMPLETED,) → True
    usage: Optional[TaskUsage]           # token/tool 消耗统计
    duration_ms: int                     # 执行时长

@dataclass(frozen=True)
class TaskUsage:
    tool_uses: int
    total_tokens: int
    duration_ms: int
```

对比当前三种模式的差异：

| 维度 | LocalAgent | RemoteAgent | InProcessTeammate |
|------|-----------|-------------|-------------------|
| 结果类型 | `AgentToolResult` | SDK raw message | `AgentToolResult` |
| 错误表示 | `error: string` | exception from HTTP poll | `error?: string` |
| 进度上报 | `AgentProgress` | event stream fragment | `AgentProgress` |
| 取消机制 | `AbortController` | session cancel API | `AbortController` + `shutdownRequested` |

统一后，主循环只需处理 `TaskResult` 一个类型，无需针对每种任务做分支。

### 9.2 统一错误处理策略

| 错误类别 | LocalAgent | RemoteAgent | InProcessTeammate | 统一后行为 |
|---------|-----------|-------------|-------------------|-----------|
| 超时 | 无显式超时 | HTTP request timeout | 无显式超时 | 统一通过 `task_timeout_seconds` 配置项控制，超时时 status→FAILED，error="timeout" |
| Budget 耗尽 | 无检查 | 无检查 | 无检查 | 在轮询回调中检查预算，接近上限时 status→KILLED，error="budget_exceeded" |
| 进程崩溃 | subprocess exit code ≠ 0 | HTTP 5xx | Python uncaught exception | 全部映射为 status→FAILED，error=规范化错误消息 |
| 权限拒绝 | Permission denied prompt | N/A | N/A | 仅 LocalAgent/InProcess 可能出现，统一映射为 status→FAILED |

**推荐实现：** 定义一个 `TaskError` 枚举类，将各类底层异常统一翻译为标准错误码：

```python
class TaskErrorType(Enum):
    TIMEOUT = "timeout"
    BUDGET_EXCEEDED = "budget_exceeded"
    PERMISSION_DENIED = "permission_denied"
    REMOTE_UNAVAILABLE = "remote_unavailable"
    LOCAL_CRASH = "local_crash"

def normalize_error(exception: Exception, context: TaskContext) -> TaskResult:
    """将所有异常统一映射为 TaskResult(status=FAILED)"""
    if isinstance(exception, TimeoutError):
        return TaskResult(..., status=TaskStatus.FAILED, error_type=TaskErrorType.TIMEOUT)
    if isinstance(exception, BudgetExceededError):
        return TaskResult(..., status=TaskStatus.FAILED, error_type=TaskErrorType.BUDGET_EXCEEDED)
    # ...
```

### 9.3 RemoteAgent 轮询容错设计

RemoteAgent 依赖 HTTP 轮询获取云端结果，需要完善的容错策略：

| 场景 | 当前行为 | 建议改进 |
|------|---------|---------|
| HTTP 5xx 瞬时错误 | 可能直接抛异常 | 指数退避重试（1s, 2s, 4s, 8s），最多 5 次 |
| 网络断开 > 30s | 未定义 | 标记 task 为半完成状态，后台保持轮询连接 |
| Claude.ai 服务不可用 | 无降级 | 自动切换到 "Local Mode"——在本地运行相同 prompt，使用相同的 Agent definition |
| 远程会话被服务端清理 | poll 返回空 | 触发 `completion_checker` 二次确认，确认后 status→FAILED |
| Token budget 接近上限 | 无检查 | 每轮轮询前检查 `cost_tracker.remaining_budget()`，< $0.50 时发送警告，< $0.10 时主动 abort |

**轮询重试伪代码：**

```python
async def poll_with_backoff(session, url, max_retries=5):
    delay = 1.0
    for attempt in range(max_retries):
        try:
            resp = await session.get(url, timeout=10)
            if resp.status == 200:
                return await resp.json()
            if resp.status >= 500:
                await asyncio.sleep(delay)
                delay *= 2  # 指数退避
                continue
            raise RemotePollError(f"HTTP {resp.status}")
        except aiohttp.ClientConnectionError:
            if attempt < max_retries - 1:
                await asyncio.sleep(delay)
                delay *= 2
            else:
                raise
    raise RemotePollError(f"Failed after {max_retries} retries")
```

### 9.4 LocalAgent 独立状态同步

LocalAgent 在主事件循环之外独立运行，其状态同步到主 loop 的机制需要明确：

| 同步通道 | 方向 | 频率 | 内容 |
|---------|------|------|------|
| `outputFile` (symlink) | Agent → Main | 写入追加 | 子代理的工具调用结果 |
| `updateTaskState(updater)` | Main → Agent state store | 每次工具调用 | progress, tool_use_count, token_count |
| `emitTaskProgress()` | Agent → SDK consumer | 周期性（~5s） | summary, recent_activities |
| `AbortController` | Main → Agent | 按需 | 取消信号 |

**关键设计约束：**
1. LocalAgent 的 `readFileState` 必须是主状态的浅克隆（shallow clone），修改文件后不反向同步到主状态，避免写冲突
2. 状态更新必须通过 `setAppState(prev => updated)` 不可变模式，禁止直接修改 `prev.tasks[taskId]`
3. `outputFile` 读取必须基于 `outputOffset` 增量读取，防止重复消费

---

## 10. 源码索引

| 组件 | 文件路径 |
|------|----------|
| Task 基类接口 | `src/Task.ts` |
| TaskType/TaskStatus 定义 | `src/Task.ts:6-29` |
| TaskStateBase 类型 | `src/Task.ts:45-57` |
| Task 基类 | `src/Task.ts:72-76` |
| 任务注册框架 | `src/utils/task/framework.ts:77-100` |
| LocalAgentTask 状态 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx:116-148` |
| registerAsyncAgent | `src/tasks/LocalAgentTask/LocalAgentTask.tsx:466-515` |
| RemoteAgentTask 状态 | `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:22-59` |
| InProcessTeammateTask 状态 | `src/tasks/InProcessTeammateTask/types.ts:22-76` |
| TeammateIdentity | `src/tasks/InProcessTeammateTask/types.ts:13-20` |
| AgentProgress | `src/tasks/LocalAgentTask/LocalAgentTask.tsx:33-39` |
| TASK_ID_ALPHABET | `src/Task.ts:96` |

---

*基于 Claude Code v2.1.88 源代码分析*
