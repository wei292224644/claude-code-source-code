# Layer 11: AUTONOMOUS AGENTS — 自主代理

## 概述

**AUTONOMOUS AGENTS** 是自驱动执行系统，支持无需用户输入的自主运行。包括 SDK/Headless 模式、后台任务调度、自动梦之记忆整合等。位于 `src/QueryEngine.ts`、`src/utils/cronScheduler.ts`、`src/services/autoDream/`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 查询引擎 | `src/QueryEngine.ts` |
| 代理执行 | `src/tools/AgentTool/runAgent.ts` |
| 异步代理生命周期 | `src/tools/AgentTool/agentToolUtils.ts` |
| 本地代理任务 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` |
| 梦之任务 | `src/tasks/DreamTask/DreamTask.ts` |
| Cron 调度器 | `src/utils/cronScheduler.ts` |
| Cron 任务 | `src/utils/cronTasks.ts` |
| 定时任务钩子 | `src/hooks/useScheduledTasks.ts` |
| 自动做梦 | `src/services/autoDream/autoDream.ts` |
| Headless 打印 | `src/cli/print.ts` |
| SDK 类型 | `src/entrypoints/agentSdkTypes.ts` |
| SDK 核心模式 | `src/entrypoints/sdk/coreSchemas.ts` |
| 协调器模式 | `src/coordinator/coordinatorMode.ts` |

---

## QueryEngine - SDK/Headless 核心

### QueryEngine 类定义

```typescript
// src/QueryEngine.ts:184-207
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
}
```

### QueryEngineConfig 配置

```typescript
// src/QueryEngine.ts:130-173
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  snipReplay?: (yieldedSystemMsg: Message, store: Message[]) => {...}
}
```

### submitMessage 核心方法

```typescript
// src/QueryEngine.ts:209-1168
// AsyncGenerator 实现，yield SDK 消息
async function* submitMessage(): AsyncGenerator<Message, void> {
  // 拥有查询生命周期和会话状态
  // 跨回合持久化状态（消息、文件缓存、使用量）
}
```

---

## SDK vs CLI 模式对比

| 方面 | SDK/Headless 模式 | CLI/REPL 模式 |
|------|-------------------|---------------|
| 入口 | `QueryEngine.submitMessage()` | `REPL.tsx` React 组件 |
| UI 回调 | `undefined`（无 TUI） | 完整 React UI |
| 状态 | `headlessStore` | `AppStateStore` |
| 调度器 | 直接 `createCronScheduler()` | `useScheduledTasks` 钩子 |
| 输入 | 通过 `enqueue()` + `run()` | 终端输入处理器 |
| 输出 | 流式 `AsyncGenerator` | StructuredIO + ink 渲染 |

---

## Cron 调度器

### CronScheduler 创建

```typescript
// src/utils/cronScheduler.ts:142-531
export function createCronScheduler(options: CronSchedulerOptions): CronScheduler {
  // ...
}

// CronSchedulerOptions 定义
type CronSchedulerOptions = {
  onFire: (prompt: string) => void
  isLoading: () => boolean
  assistantMode?: boolean
  onFireTask?: (task: CronTask) => void
  onMissed?: (tasks: CronTask[]) => void
  dir?: string  // daemon 调用者使用 - 绕过引导状态
  lockIdentity?: string  // 稳定进程 UUID
  filter?: (t: CronTask) => boolean  // 按任务过滤
}
```

### CronTask 结构

```typescript
// src/utils/cronTasks.ts
export type CronTask = {
  id: string
  cron: string
  prompt: string
  createdAt: number
  recurring?: boolean
  permanent?: boolean  // 永不过期
  agentId?: string    // 路由到特定队友
}
```

### 调度器锁机制

```typescript
// src/utils/cronScheduler.ts:396-460
// enable() 获取每目录调度器锁（基于 PID）
// 防止多个 Claude 实例重复触发
```

### Headless 模式调度

```typescript
// src/cli/print.ts:2696-2734
let cronScheduler: CronScheduler | null = null
if (feature('AGENT_TRIGGERS') && cronSchedulerModule && cronGate?.isKairosCronEnabled()) {
  cronScheduler = cronSchedulerModule.createCronScheduler({
    onFire: prompt => {
      if (inputClosed) return
      enqueue({
        mode: 'prompt',
        value: prompt,
        uuid: randomUUID(),
        priority: 'later',
        isMeta: true,
        workload: WORKLOAD_CRON,
      })
      void run()
    },
    isLoading: () => running || inputClosed,
  })
  cronScheduler.start()
}
```

---

## 自动做梦 (Auto-Dream)

### 自动做梦配置

```typescript
// src/services/autoDream/autoDream.ts:54-66
const DEFAULTS: AutoDreamConfig = {
  minHours: 24,
  minSessions: 5,
}
```

### 触发条件（按成本排序）

```
1. 时间门：hours since lastConsolidatedAt >= minHours
2. 会话门：transcript count >= minSessions
3. 锁门：无其他进程正在整合
```

### initAutoDream 实现

```typescript
// src/services/autoDream/autoDream.ts:122-273
export function initAutoDream(): void {
  runner = async function runAutoDream(context, appendSystemMessage) {
    // 时间门
    const hoursSince = (Date.now() - lastAt) / 3_600_000
    if (!force && hoursSince < cfg.minHours) return

    // 会话门
    sessionIds = await listSessionsTouchedSince(lastAt)
    if (!force && sessionIds.length < cfg.minSessions) return

    // 锁获取
    priorMtime = await tryAcquireConsolidationLock()

    // 使用 /dream 提示词运行分叉代理
    const result = await runForkedAgent({
      promptMessages: [createUserMessage({ content: prompt })],
      cacheSafeParams: createCacheSafeParams(context),
      canUseTool: createAutoMemCanUseTool(memoryRoot),
      querySource: 'auto_dream',
      forkLabel: 'auto_dream',
    })
  }
}
```

---

## 异步代理生命周期

### registerAsyncAgent

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
}): LocalAgentTaskState {
  // 创建中止控制器 - 如果提供了父级，创建子级
  const abortController = parentAbortController
    ? createChildAbortController(parentAbortController)
    : createAbortController()

  const taskState: LocalAgentTaskState = {
    ...createTaskStateBase(agentId, 'local_agent', description, toolUseId),
    type: 'local_agent',
    status: 'running',
    isBackgrounded: true,  // 立即后台化
    pendingMessages: [],
    retain: false,
    diskLoaded: false
  }

  registerTask(taskState, setAppState)
  return taskState
}
```

### 自动后台化超时

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx:580-608
if (autoBackgroundMs !== undefined && autoBackgroundMs > 0) {
  const timer = setTimeout((setAppState, agentId) => {
    setAppState(prev => {
      const prevTask = prev.tasks[agentId]
      if (!isLocalAgentTask(prevTask) || prevTask.isBackgrounded) return prev
      return { ...prev, tasks: { ...prev.tasks, [agentId]: { ...prevTask, isBackgrounded: true } } }
    })
  }, autoBackgroundMs, setAppState, agentId)
}
```

### runAsyncAgentLifecycle

```typescript
// src/tools/AgentTool/agentToolUtils.ts:508-637
export async function runAsyncAgentLifecycle({
  taskId,
  abortController,
  makeStream,
  metadata,
  description,
  toolUseContext,
  rootSetAppState,
  agentIdForCleanup,
  enableSummarization,
  getWorktreeResult,
}): Promise<void> {
  const tracker = createProgressTracker()

  // 进度跟踪循环
  for await (const message of makeStream(onCacheSafeParams)) {
    agentMessages.push(message)
    // 更新 UI 任务状态
    updateAsyncAgentProgress(taskId, getProgressUpdate(tracker), rootSetAppState)
  }

  // 完成通知
  completeAsyncAgent(agentResult, rootSetAppState)
  enqueueAgentNotification({ taskId, description, status: 'completed', ... })
}
```

---

## 循环中断机制

### 最大回合限制

```typescript
// src/QueryEngine.ts:841-874
if (message.attachment.type === 'max_turns_reached') {
  yield {
    type: 'result',
    subtype: 'error_max_turns',
    errors: [`Reached maximum number of turns (${message.attachment.maxTurns})`],
  }
  return
}
```

### 预算控制

```typescript
// src/QueryEngine.ts:972-1002
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield {
    type: 'result',
    subtype: 'error_max_budget_usd',
    errors: [`Reached maximum budget ($${maxBudgetUsd})`],
  }
  return
}
```

### 结构化输出重试限制

```typescript
// src/QueryEngine.ts:1005-1048
const maxRetries = parseInt(process.env.MAX_STRUCTURED_OUTPUT_RETRIES || '5', 10)
if (callsThisQuery >= maxRetries) {
  yield { type: 'result', subtype: 'error_max_structured_output_retries', ... }
  return
}
```

### AbortController

```typescript
// src/utils/abortController.js
export function createChildAbortController(parent: AbortController): AbortController {
  const controller = new AbortController()
  parent.signal.addEventListener('abort', () => controller.abort())
  return controller
}
```

---

## DreamTask 注册

```typescript
// src/tasks/DreamTask/DreamTask.ts:52-74
export function registerDreamTask(setAppState: SetAppState, opts: {...}): string {
  const task: DreamTaskState = {
    ...createTaskStateBase(id, 'dream', 'dreaming'),
    type: 'dream',
    status: 'running',
    phase: 'starting',
    sessionsReviewing: opts.sessionsReviewing,
    filesTouched: [],
    turns: [],
    abortController: opts.abortController,
  }
  registerTask(task, setAppState)
  return id
}
```

---

## 自主代理工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     自主代理执行流程                                  │
│                                                                     │
│  1. 定时触发 (Cron)                                                  │
│     cronScheduler.check() → onFire(prompt)                         │
│                                                                     │
│  2. 自动做梦触发                                                     │
│     initAutoDream() → 时间门 + 会话门 + 锁门                        │
│                                                                     │
│  3. 异步代理启动                                                     │
│     registerAsyncAgent() → 后台任务状态                             │
│                                                                     │
│  4. 执行循环                                                        │
│     runAgent(isAsync=true) → 独立 AbortController                   │
│                                                                     │
│  5. 进度跟踪                                                         │
│     runAsyncAgentLifecycle() → 更新任务进度                         │
│                                                                     │
│  6. 完成通知                                                         │
│     completeAsyncAgent() → enqueueAgentNotification()               │
│                                                                     │
│  循环中断条件:                                                       │
│  - maxTurns 达到                                                    │
│  - maxBudgetUsd 达到                                               │
│  - AbortController 信号                                             │
│  - 结构化输出重试超限                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计

### 1. 无 UI 回调

```typescript
// SDK 模式下，UI 回调为 undefined
// setToolJSX, setStreamMode 等不设置
```

### 2. 进程隔离的 AbortController

```typescript
// 异步代理创建独立的 AbortController
// 不关联父级信号，可独立中止
const agentAbortController = isAsync
  ? new AbortController()
  : toolUseContext.abortController
```

### 3. Cron 锁机制

```typescript
// 每目录调度器锁（基于 PID）
// 防止多个 Claude 实例重复触发同一任务
```

### 4. Auto-Dream 门控

```typescript
// 按成本排序的触发条件
// 1. 时间门（最便宜）
// 2. 会话门
// 3. 锁门（最贵）
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| QueryEngine | `src/QueryEngine.ts:184-207` |
| submitMessage | `src/QueryEngine.ts:209-1168` |
| QueryEngineConfig | `src/QueryEngine.ts:130-173` |
| maxTurns 处理 | `src/QueryEngine.ts:841-874` |
| maxBudgetUsd | `src/QueryEngine.ts:972-1002` |
| createCronScheduler | `src/utils/cronScheduler.ts:142-531` |
| CronTask | `src/utils/cronTasks.ts` |
| initAutoDream | `src/services/autoDream/autoDream.ts:122-273` |
| registerAsyncAgent | `src/tasks/LocalAgentTask/LocalAgentTask.tsx:466-515` |
| runAsyncAgentLifecycle | `src/tools/AgentTool/agentToolUtils.ts:508-637` |
| registerDreamTask | `src/tasks/DreamTask/DreamTask.ts:52-74` |
| runAgent (isAsync) | `src/tools/AgentTool/runAgent.ts:248-329` |
| createChildAbortController | `src/utils/abortController.js` |
| useScheduledTasks | `src/hooks/useScheduledTasks.ts:40-127` |
| Headless Cron | `src/cli/print.ts:2696-2734` |

---

*基于 Claude Code v2.1.88 源代码分析*
