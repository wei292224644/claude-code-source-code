# Layer 09: AGENT TEAMS — 多代理团队

## 概述

**AGENT TEAMS** 是多代理协作系统。多个 AI 代理可以在同一团队中分工合作，通过邮箱系统进行进程间通信。位于 `src/tools/TeamCreateTool/`、`src/tools/TeamDeleteTool/`、`src/utils/swarm/`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 团队创建 | `src/tools/TeamCreateTool/TeamCreateTool.ts` |
| 团队删除 | `src/tools/TeamDeleteTool/TeamDeleteTool.ts` |
| 进程内队友任务 | `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` |
| 团队辅助函数 | `src/utils/swarm/teamHelpers.ts` |
| 邮箱系统 | `src/utils/teammateMailbox.ts` |
| 队友上下文 | `src/utils/teammateContext.ts` |
| 队友工具函数 | `src/utils/teammate.ts` |
| 进程内生成 | `src/utils/swarm/spawnInProcess.ts` |
| 多代理生成 | `src/tools/shared/spawnMultiAgent.ts` |
| 团队初始化钩子 | `src/utils/swarm/teammateInit.ts` |

---

## 团队核心数据结构

### TeamFile 结构

```typescript
// src/utils/swarm/teamHelpers.ts:64-90
export type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    prompt?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

### 团队存储结构

```
~/.claude/teams/<team-name>/
├── config.json              # 团队配置
└── inboxes/                 # 邮箱目录
    ├── leader.json          # 队长收件箱
    └── <teammate>.json      # 队友收件箱
```

---

## TeamCreateTool - 创建团队

### 核心实现

```typescript
// src/tools/TeamCreateTool/TeamCreateTool.ts:74-240
export const TeamCreateTool: Tool<InputSchema, Output> = buildTool({
  name: TEAM_CREATE_TOOL_NAME,
  searchHint: 'create a multi-agent swarm team',
  maxResultSizeChars: 100_000,
  shouldDefer: true,

  async call(input, context) {
    // 1. 检查队长是否已有团队（每个队长只能有一个团队）
    // 2. 生成唯一团队名（避免冲突）
    // 3. 创建团队文件 ~/.claude/teams/{team-name}/config.json
    // 4. 重置/创建任务列表目录
    // 5. 更新 AppState 的 teamContext
  },
})
```

### 团队工作流程

```
创建团队 → 创建任务 → 生成队友 → 分配任务 → 关闭
```

---

## TeamDeleteTool - 删除团队

### 核心实现

```typescript
// src/tools/TeamDeleteTool/TeamDeleteTool.ts:32-139
export const TeamDeleteTool: Tool<InputSchema, Output> = buildTool({
  name: TEAM_DELETE_TOOL_NAME,

  async call(_input, context) {
    // 1. 读取团队文件检查活跃成员
    // 2. 如果有活跃成员则失败（必须先关闭）
    // 3. 清理团队目录和 git worktree
    // 4. 清除 AppState 中的 teamContext 和收件箱
  },
})
```

### 安全检查

```typescript
// src/tools/TeamDeleteTool/TeamDeleteTool.ts:85-98
// 过滤掉队长，只计算非队长成员
const nonLeadMembers = teamFile.members.filter(
  m => m.name !== TEAM_LEAD_NAME,
)
// 将真正活跃的成员与空闲/死亡的成员分开
// isActive === false 的成员是空闲的（完成回合或崩溃）
const activeMembers = nonLeadMembers.filter(m => m.isActive !== false)

if (activeMembers.length > 0) {
  return {
    data: {
      success: false,
      message: `Cannot cleanup team with ${activeMembers.length} active member(s): ${memberNames}. Use requestShutdown to gracefully terminate teammates first.`,
    }
  }
}
```

---

## InProcessTeammateTask - 进程内队友任务

### 任务接口

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

### 任务状态类型

```typescript
// src/tasks/InProcessTeammateTask/types.ts:22-76
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity  // agentId, agentName, teamName, color, planModeRequired, parentSessionId
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController  // 运行时 - 终止整个队友
  currentWorkAbortController?: AbortController  // 运行时 - 中止当前回合
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  messages?: Message[]  // 缩放视图的对话历史
  inProgressToolUseIDs?: Set<string>
  pendingUserMessages: string[]
  spinnerVerb?: string
  pastTenseVerb?: string
  isIdle: boolean
  shutdownRequested: boolean
  onIdleCallbacks?: Array<() => void>  // 通知队友空闲时的回调
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

### 关键辅助函数

```typescript
// src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:35-125
// 请求关闭队友
export function requestTeammateShutdown(taskId: string, setAppState: SetAppState): void

// 向队友对话历史追加消息
export function appendTeammateMessage(taskId: string, message: Message, setAppState: SetAppState): void

// 向队友待处理队列注入用户消息
export function injectUserMessageToTeammate(taskId: string, message: string, setAppState: SetAppState): void

// 按 agent ID 查找队友任务
export function findTeammateTaskByAgentId(agentId: string, tasks: Record<string, TaskStateBase>): InProcessTeammateTaskState | undefined

// 获取所有进程内队友任务
export function getAllInProcessTeammateTasks(tasks: Record<string, TaskStateBase>): InProcessTeammateTaskState[]

// 按字母顺序获取运行中的队友
export function getRunningTeammatesSorted(tasks: Record<string, TaskStateBase>): InProcessTeammateTaskState[]
```

### 消息上限

```typescript
// src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:100-121
// BQ 分析显示：500+ 回合的会话每个代理约 20MB RSS
// 9a990de8 的 whale 会话在 2 分钟内启动了 292 个代理，达到 36.8GB
export const TEAMMATE_MESSAGES_UI_CAP = 50

export function appendCappedMessage<T>(prev: readonly T[] | undefined, item: T): T[]
```

---

## 队友上下文隔离

### TeammateContext 类型

```typescript
// src/utils/teammateContext.ts:1-96
export type TeammateContext = {
  agentId: string           // 例如 "researcher@my-team"
  agentName: string         // 例如 "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string   // 队长的会话 ID
  isInProcess: true
  abortController: AbortController
}

// 使用 AsyncLocalStorage 实现并发执行隔离
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

export function getTeammateContext(): TeammateContext | undefined
export function runWithTeammateContext<T>(context: TeammateContext, fn: () => T): T
export function isInProcessTeammate(): boolean
```

### 身份解析优先级

```typescript
// src/utils/teammate.ts:11-118
// 身份解析优先级：
// 1. AsyncLocalStorage（进程内队友）- 通过 teammateContext.ts
// 2. dynamicTeamContext（通过 CLI 参数的 tmux 队友）
// 3. 环境变量

export function getAgentId(): string | undefined
export function getAgentName(): string | undefined
export function getTeamName(teamContext?: { teamName: string }): string | undefined
export function isTeammate(): boolean
export function isTeamLead(teamContext: { leadAgentId: string }): boolean
```

---

## 邮箱系统

### 收件箱路径结构

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

### 核心邮箱函数

```typescript
// src/utils/teammateMailbox.ts:84-192
export async function readMailbox(agentName: string, teamName?: string): Promise<TeammateMessage[]>
export async function readUnreadMessages(agentName: string, teamName?: string): Promise<TeammateMessage[]>
export async function writeToMailbox(
  recipientName: string,
  message: Omit<TeammateMessage, 'read'>,
  teamName?: string,
): Promise<void>
// 使用文件锁和重试处理并发访问
```

### 消息类型

```typescript
// src/utils/teammateMailbox.ts:43-50, 393-1062
export type TeammateMessage = {
  from: string
  text: string
  timestamp: string
  read: boolean
  color?: string
  summary?: string
}
```

### 结构化协议消息

| 消息类型 | 描述 |
|---------|------|
| `IdleNotificationMessage` | 队友空闲时发送 |
| `PermissionRequestMessage` | 工具权限请求 |
| `PlanApprovalRequestMessage` | 计划模式审批请求 |
| `ShutdownRequestMessage` | 关闭请求 |
| `TaskAssignmentMessage` | 任务分配通知 |
| `ModeSetRequestMessage` | 权限模式变更 |

### 协议检测

```typescript
// src/utils/teammateMailbox.ts:1073-1095
export function isStructuredProtocolMessage(messageText: string): boolean {
  // 返回 true 的消息应由 useInboxPoller 路由
  // 而不是作为原始 LLM 上下文消费
}
```

---

## 进程内队友生成

### spawnInProcessTeammate

```typescript
// src/utils/swarm/spawnInProcess.ts:104-216
export async function spawnInProcessTeammate(
  config: InProcessSpawnConfig,
  context: SpawnContext,
): Promise<InProcessSpawnOutput> {
  // 1. 生成确定性 agent ID: formatAgentId(name, teamName)
  // 2. 创建独立的 AbortController（不关联父级）
  // 3. 创建 TeammateIdentity 和 TeammateContext
  // 4. 注册清理处理器
  // 5. 在 AppState 中注册任务
  // 6. 返回供执行使用的生成结果
}
```

### killInProcessTeammate

```typescript
// src/utils/swarm/spawnInProcess.ts:227-328
export function killInProcessTeammate(
  taskId: string,
  setAppState: SetAppStateFn,
): boolean {
  // 1. 中止控制器
  // 2. 调用清理处理器
  // 3. 调用待处理的空闲回调以解除等待者
  // 4. 从 teamContext.teammates 移除
  // 5. 更新任务状态为 'killed'
  // 6. 从团队文件移除
  // 7. 从 perfetto 注册表中驱逐
}
```

---

## 团队辅助函数

### cleanupTeamDirectories

```typescript
// src/utils/swarm/teamHelpers.ts:641-683
export async function cleanupTeamDirectories(teamName: string): Promise<void> {
  // 1. 删除前读取团队文件获取 worktree 路径
  // 2. 先清理 git worktree
  for (const worktreePath of worktreePaths) {
    await destroyWorktree(worktreePath)
  }
  // 3. 清理团队目录 ~/.claude/teams/{team-name}/
  // 4. 清理任务目录 ~/.claude/tasks/{taskListId}/
}
```

### cleanupSessionTeams

```typescript
// src/utils/swarm/teamHelpers.ts:576-590
export async function cleanupSessionTeams(): Promise<void> {
  // 在非正常退出时调用（SIGINT/SIGTERM）
  // 杀死孤立的队友 pane 并清理目录
  await Promise.allSettled(teams.map(name => killOrphanedTeammatePanes(name)))
  await Promise.allSettled(teams.map(name => cleanupTeamDirectories(name)))
}
```

---

## 队友初始化钩子

```typescript
// src/utils/swarm/teammateInit.ts:28-129
export function initializeTeammateHooks(
  setAppState: (updater: (prev: AppState) => AppState) => void,
  sessionId: string,
  teamInfo: { teamName: string; agentId: string; agentName: string },
): void {
  // 1. 读取团队文件获取队长信息
  // 2. 应用团队范围的允许路径
  // 3. 注册 Stop 钩子：
  //    - 在团队配置中标记队友为空闲
  //    - 通过邮箱向团队队长发送空闲通知
}
```

---

## 多代理生成

### 生成处理器

```typescript
// src/tools/shared/spawnMultiAgent.ts
// handleSpawnSplitPane - tmux/iTerm2 分屏视图
// handleSpawnSeparateWindow - 传统独立窗口
// handleSpawnInProcess - 通过 AsyncLocalStorage 的进程内队友
// handleSpawn - 主调度器，带回退逻辑
```

### 生成输出

```typescript
// src/tools/shared/spawnMultiAgent.ts:107-120
export type SpawnOutput = {
  teammate_id: string
  agent_id: string
  agent_type?: string
  model?: string
  name: string
  color?: string
  tmux_session_name: string
  tmux_window_name: string
  tmux_pane_id: string
  team_name?: string
  is_splitpane?: boolean
  plan_mode_required?: boolean
}
```

---

## AppState 团队上下文

```typescript
// src/state/AppStateStore.ts:323-333
teamContext?: {
  teamName: string
  teamFilePath: string
  leadAgentId: string
  selfAgentId?: string    // swarm 成员自己的 ID
  selfAgentName?: string  // swarm 成员的名字
  isLeader?: boolean      // 如果是团队队长则为 true
  selfAgentColor?: string
  teammates: {
    [agentId: string]: {
      name: string
      agentType?: string
      color?: string
      tmuxSessionName: string
      tmuxPaneId: string
      cwd: string
      spawnedAt: number
    }
  }
}
```

---

## 团队工作流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     多代理团队工作流程                                 │
│                                                                     │
│  1. TeamCreateTool(name="my-team")                                │
│     - 创建 ~/.claude/teams/my-team/config.json                    │
│     - 初始化 teamContext                                           │
│                                                                     │
│  2. AgentTool 创建队友 (in_process 或 tmux)                        │
│     - spawnInProcessTeammate() 或 tmux spawn                      │
│     - 注册到 teamContext.teammates                                │
│                                                                     │
│  3. 队友执行任务                                                    │
│     - 通过 AsyncLocalStorage 隔离上下文                            │
│     - 通过邮箱系统与队长通信                                        │
│                                                                     │
│  4. 任务分配                                                        │
│     - TaskCreateTool 创建任务，owner 设为队友                       │
│     - 队友通过 TaskUpdateTool 更新状态                             │
│                                                                     │
│  5. 通信                                                            │
│     - writeToMailbox() / readMailbox()                            │
│     - 空闲通知、权限请求、计划审批等                                 │
│                                                                     │
│  6. 关闭                                                            │
│     - sendShutdownRequestToMailbox()                              │
│     - TeamDeleteTool 清理团队                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 进程内 vs tmux 队友

| 特性 | In-Process | tmux |
|------|-------------|------|
| 执行方式 | AsyncLocalStorage 隔离 | 独立进程 |
| 通信延迟 | 无（内存共享） | 文件系统邮箱 |
| 资源消耗 | 共享进程 | 独立进程 |
| 容错性 | 一个崩溃可能影响全部 | 隔离性好 |
| 使用场景 | 快速协作、轻量任务 | 长期运行、重型任务 |

---

## 关键设计

### 1. AsyncLocalStorage 隔离

```typescript
// 进程内队友使用 AsyncLocalStorage 实现并发隔离
const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()

export function runWithTeammateContext<T>(context: TeammateContext, fn: () => T): T {
  return teammateContextStorage.run(context, fn)
}
```

### 2. 邮箱文件锁

```typescript
// 并发访问使用重试机制
const LOCK_OPTIONS = {
  retries: {
    retries: 30,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

### 3. 消息上限保护

```typescript
// 防止内存泄漏
export const TEAMMATE_MESSAGES_UI_CAP = 50
// 超过后丢弃最旧的消息
```

### 4. 非正常退出清理

```typescript
// SIGINT/SIGTERM 时自动清理孤立团队
cleanupSessionTeams()
// 杀死 orphaned panes 并清理目录
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| TeamCreateTool | `src/tools/TeamCreateTool/TeamCreateTool.ts:74-240` |
| TeamDeleteTool | `src/tools/TeamDeleteTool/TeamDeleteTool.ts:32-139` |
| InProcessTeammateTask | `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:24-30` |
| InProcessTeammateTaskState | `src/tasks/InProcessTeammateTask/types.ts:22-76` |
| TeamFile | `src/utils/swarm/teamHelpers.ts:64-90` |
| TeammateContext | `src/utils/teammateContext.ts:1-96` |
| readMailbox | `src/utils/teammateMailbox.ts:84-192` |
| writeToMailbox | `src/utils/teammateMailbox.ts:84-192` |
| spawnInProcessTeammate | `src/utils/swarm/spawnInProcess.ts:104-216` |
| killInProcessTeammate | `src/utils/swarm/spawnInProcess.ts:227-328` |
| cleanupTeamDirectories | `src/utils/swarm/teamHelpers.ts:641-683` |
| initializeTeammateHooks | `src/utils/swarm/teammateInit.ts:28-129` |
| handleSpawnInProcess | `src/tools/shared/spawnMultiAgent.ts:840-1032` |
| waitForTeammatesToBecomeIdle | `src/utils/teammate.ts:238-292` |

---

*基于 Claude Code v2.1.88 源代码分析*
