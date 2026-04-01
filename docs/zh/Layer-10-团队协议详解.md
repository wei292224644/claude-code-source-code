# Layer 10: TEAM PROTOCOLS — 团队协议

## 概述

**TEAM PROTOCOLS** 是多代理团队间的协调机制。包括权限传播、计划审批、消息路由、领导选举等协议。位于 `src/utils/swarm/`、`src/hooks/useInboxPoller.ts`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 权限同步 | `src/utils/swarm/permissionSync.ts` |
| 团队辅助函数 | `src/utils/swarm/teamHelpers.ts` |
| 邮箱系统 | `src/utils/teammateMailbox.ts` |
| 收件箱轮询 | `src/hooks/useInboxPoller.ts` |
| 权限轮询 | `src/hooks/useSwarmPermissionPoller.ts` |
| 进程内运行器 | `src/utils/swarm/inProcessRunner.ts` |
| 队友初始化 | `src/utils/swarm/teammateInit.ts` |
| 队友工具函数 | `src/utils/teammate.ts` |
| 协调器模式 | `src/coordinator/coordinatorMode.ts` |
| 团队常量 | `src/utils/swarm/constants.ts` |

---

## 团队常量

```typescript
// src/utils/swarm/constants.ts:1-33
export const TEAM_LEAD_NAME = 'team-lead'
export const SWARM_SESSION_NAME = 'claude-swarm'
export const SWARM_VIEW_WINDOW_NAME = 'swarm-view'
export const TMUX_COMMAND = 'tmux'
export const HIDDEN_SESSION_NAME = 'claude-hidden'

// 队友配置的环境变量
export const TEAMMATE_COMMAND_ENV_VAR = 'CLAUDE_CODE_TEAMMATE_COMMAND'
export const TEAMMATE_COLOR_ENV_VAR = 'CLAUDE_CODE_AGENT_COLOR'
export const PLAN_MODE_REQUIRED_ENV_VAR = 'CLAUDE_CODE_PLAN_MODE_REQUIRED'
```

---

## 权限传播协议

### 团队允许路径

```typescript
// src/utils/swarm/teammateInit.ts:44-79
// 队友初始化时应用团队范围的允许路径
if (teamFile.teamAllowedPaths && teamFile.teamAllowedPaths.length > 0) {
  for (const allowedPath of teamFile.teamAllowedPaths) {
    const ruleContent = allowedPath.path.startsWith('/')
      ? `/${allowedPath.path}/**`
      : `${allowedPath.path}/**`

    setAppState(prev => ({
      ...prev,
      toolPermissionContext: applyPermissionUpdate(
        prev.toolPermissionContext,
        {
          type: 'addRules',
          rules: [{ toolName: allowedPath.toolName, ruleContent }],
          behavior: 'allow',
          destination: 'session',
        },
      ),
    }))
  }
}
```

### 权限请求流程 (队友 → 队长)

```typescript
// src/utils/swarm/permissionSync.ts:676-722
export async function sendPermissionRequestViaMailbox(
  request: SwarmPermissionRequest,
): Promise<boolean> {
  const leaderName = await getLeaderName(request.teamName)
  if (!leaderName) return false

  const message = createPermissionRequestMessage({
    request_id: request.id,
    agent_id: request.workerName,
    tool_name: request.toolName,
    tool_use_id: request.toolUseId,
    description: request.description,
    input: request.input,
    permission_suggestions: request.permissionSuggestions,
  })

  await writeToMailbox(
    leaderName,
    {
      from: request.workerName,
      text: jsonStringify(message),
      timestamp: new Date().toISOString(),
      color: request.workerColor,
    },
    request.teamName,
  )
  return true
}
```

### 权限响应流程 (队长 → 队友)

```typescript
// src/utils/swarm/permissionSync.ts:734-783
export async function sendPermissionResponseViaMailbox(
  workerName: string,
  resolution: PermissionResolution,
  requestId: string,
  teamName?: string,
): Promise<boolean> {
  const message = createPermissionResponseMessage({
    request_id: requestId,
    subtype: resolution.decision === 'approved' ? 'success' : 'error',
    error: resolution.feedback,
    updated_input: resolution.updatedInput,
    permission_updates: resolution.permissionUpdates,
  })

  await writeToMailbox(
    workerName,
    {
      from: senderName,
      text: jsonStringify(message),
      timestamp: new Date().toISOString(),
    },
    team,
  )
  return true
}
```

### 进程内队友权限处理

```typescript
// src/utils/swarm/inProcessRunner.ts:195-334
// 进程内队友使用队长 UI 队列处理权限
const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()

if (setToolUseConfirmQueue) {
  return new Promise<PermissionDecision>(resolve => {
    setToolUseConfirmQueue(queue => [
      ...queue,
      {
        assistantMessage,
        tool: tool as Tool,
        description,
        input,
        toolUseContext,
        toolUseID,
        workerBadge: identity.color
          ? { name: identity.agentName, color: identity.color }
          : undefined,
        onAllow(updatedInput, permissionUpdates, feedback) {
          // 持久化权限更新并写回队长上下文
          if (permissionUpdates.length > 0) {
            const setToolPermissionContext = getLeaderSetToolPermissionContext()
            if (setToolPermissionContext) {
              const updatedContext = applyPermissionUpdates(...)
              setToolPermissionContext(updatedContext, { preserveMode: true })
            }
          }
          resolve({ behavior: 'allow', updatedInput, ... })
        },
        onReject(feedback) {
          resolve({ behavior: 'ask', message: SUBAGENT_REJECT_MESSAGE })
        },
      },
    ])
  })
}
```

### CLI 标志继承

```typescript
// src/tools/shared/spawnMultiAgent.ts:208-260
function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []

  // 传播权限模式给队友，但计划模式要求时除外
  if (planModeRequired) {
    // 要求计划模式时不继承绕过权限
  } else if (permissionMode === 'bypassPermissions' || getSessionBypassPermissionsMode()) {
    flags.push('--dangerously-skip-permissions')
  } else if (permissionMode === 'acceptEdits') {
    flags.push('--permission-mode acceptEdits')
  } else if (permissionMode === 'auto') {
    flags.push('--permission-mode auto')
  }
  return flags.join(' ')
}
```

---

## 计划审批协议

### 计划审批请求消息

```typescript
// src/utils/teammateMailbox.ts:684-697
export const PlanApprovalRequestMessageSchema = lazySchema(() =>
  z.object({
    type: z.literal('plan_approval_request'),
    from: z.string(),
    timestamp: z.string(),
    planFilePath: z.string(),
    planContent: z.string(),
    requestId: z.string(),
  }),
)

// 计划审批响应消息
export const PlanApprovalResponseMessageSchema = lazySchema(() =>
  z.object({
    type: z.literal('plan_approval_response'),
    requestId: z.string(),
    approved: z.boolean(),
    feedback: z.string().optional(),
    timestamp: z.string(),
    permissionMode: PermissionModeSchema().optional(),
  }),
)
```

### 队友端计划审批处理

```typescript
// src/hooks/useInboxPoller.ts:156-196
// 检查计划审批响应并过渡出计划模式
if (isTeammate() && isPlanModeRequired()) {
  for (const msg of unread) {
    const approvalResponse = isPlanApprovalResponse(msg.text)
    // 验证消息来自队长，防止队友伪造审批
    if (approvalResponse && msg.from === 'team-lead') {
      if (approvalResponse.approved) {
        const targetMode = approvalResponse.permissionMode ?? 'default'
        setAppState(prev => ({
          ...prev,
          toolPermissionContext: applyPermissionUpdate(
            prev.toolPermissionContext,
            { type: 'setMode', mode: toExternalPermissionMode(targetMode), destination: 'session' },
          ),
        }))
      }
    }
  }
}
```

### 队长自动审批

```typescript
// src/hooks/useInboxPoller.ts:599-662
// 处理计划审批请求（队长端）- 自动审批并写回响应
if (planApprovalRequests.length > 0 && isTeamLead(currentAppState.teamContext)) {
  for (const m of planApprovalRequests) {
    const parsed = isPlanApprovalRequest(m.text)
    if (!parsed) continue

    const approvalResponse = {
      type: 'plan_approval_response',
      requestId: parsed.requestId,
      approved: true,
      timestamp: new Date().toISOString(),
      permissionMode: modeToInherit,
    }

    // 写审批响应到队友收件箱
    void writeToMailbox(m.from, {
      from: TEAM_LEAD_NAME,
      text: jsonStringify(approvalResponse),
      timestamp: new Date().toISOString(),
    }, teamName)
  }
}
```

---

## 消息路由系统

### 邮箱路径结构

```typescript
// src/utils/teammateMailbox.ts:56-66
export function getInboxPath(agentName: string, teamName?: string): string {
  const team = teamName || getTeamName() || 'default'
  const inboxDir = join(getTeamsDir(), safeTeam, 'inboxes')
  return join(inboxDir, `${safeAgentName}.json`)
}
```

### 消息类型检测

```typescript
// src/utils/teammateMailbox.ts:1073-1095
export function isStructuredProtocolMessage(messageText: string): boolean {
  const parsed = jsonParse(messageText)
  const type = parsed.type
  return (
    type === 'permission_request' ||
    type === 'permission_response' ||
    type === 'sandbox_permission_request' ||
    type === 'shutdown_request' ||
    type === 'shutdown_approved' ||
    type === 'team_permission_update' ||
    type === 'mode_set_request' ||
    type === 'plan_approval_request' ||
    type === 'plan_approval_response'
  )
}
```

### 收件箱轮询消息路由

```typescript
// src/hooks/useInboxPoller.ts:204-248
const permissionRequests: TeammateMessage[] = []
const permissionResponses: TeammateMessage[] = []
const sandboxPermissionRequests: TeammateMessage[] = []
const shutdownRequests: TeammateMessage[] = []
const planApprovalRequests: TeammateMessage[] = []
const regularMessages: TeammateMessage[] = []

for (const m of unread) {
  if (isPermissionRequest(m.text)) permissionRequests.push(m)
  else if (isPermissionResponse(m.text)) permissionResponses.push(m)
  else if (isSandboxPermissionRequest(m.text)) sandboxPermissionRequests.push(m)
  else if (isShutdownRequest(m.text)) shutdownRequests.push(m)
  else if (isPlanApprovalRequest(m.text)) planApprovalRequests.push(m)
  else regularMessages.push(m)
}
```

### 队长端权限请求处理

```typescript
// src/hooks/useInboxPoller.ts:250-364
if (permissionRequests.length > 0 && isTeamLead(currentAppState.teamContext)) {
  const setToolUseConfirmQueue = getLeaderToolUseConfirmQueue()
  for (const m of permissionRequests) {
    const parsed = isPermissionRequest(m.text)
    if (setToolUseConfirmQueue) {
      const entry: ToolUseConfirm = {
        assistantMessage: createAssistantMessage({ content: '' }),
        tool,
        description: parsed.description,
        input: parsed.input,
        toolUseID: parsed.tool_use_id,
        workerBadge: { name: parsed.agent_id, color: 'cyan' },
        onAbort() {
          void sendPermissionResponseViaMailbox(...)
        },
        onAllow(updatedInput, permissionUpdates) {
          void sendPermissionResponseViaMailbox(...)
        },
      }
    }
  }
}
```

---

## 领导选举

### 团队队长检测

```typescript
// src/utils/teammate.ts:171-198
export function isTeamLead(
  teamContext: { leadAgentId: string } | undefined,
): boolean {
  if (!teamContext?.leadAgentId) return false
  const myAgentId = getAgentId()
  const leadAgentId = teamContext.leadAgentId
  // 如果我的 agent ID 匹配队长 ID，我是队长
  if (myAgentId === leadAgentId) return true
  // 向后兼容：如果没有设置 agent ID 且有团队上下文，就是创建团队的原始会话（队长）
  if (!myAgentId) return true
  return false
}
```

### 获取队长名称

```typescript
// src/utils/swarm/permissionSync.ts:651-667
export async function getLeaderName(teamName?: string): Promise<string | null> {
  const team = teamName || getTeamName()
  if (!team) return null
  const teamFile = await readTeamFileAsync(team)
  if (!teamFile) return null
  const leadMember = teamFile.members.find(m => m.agentId === teamFile.leadAgentId)
  return leadMember?.name || 'team-lead'
}
```

---

## 协调器模式

### 协调器系统提示词

```typescript
// src/coordinator/coordinatorMode.ts:111-369
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user

## 2. Your Tools
- **Agent** - Spawn a new worker
- **SendMessage** - Continue an existing worker
- **TaskStop** - Stop a running worker

## 3. Workers
When calling Agent, use subagent_type \`worker\`. Workers execute tasks autonomously.
Workers can't see your conversation. Every prompt must be self-contained.
`
}
```

### 协调器模式检测

```typescript
// src/hooks/toolPermission/handlers/coordinatorHandler.ts:36-41
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

---

## 空闲通知协议

```typescript
// src/utils/swarm/teammateInit.ts:97-129
addFunctionHook(
  setAppState,
  sessionId,
  'Stop',
  '',
  async (messages, _signal) => {
    // 1. 标记成员不活跃
    void setMemberActive(teamName, agentName, false)

    // 2. 发送空闲通知给队长
    const notification = createIdleNotification(agentName, {
      idleReason: 'available',
      summary: getLastPeerDmSummary(messages),
    })
    await writeToMailbox(leadAgentName, {
      from: agentName,
      text: jsonStringify(notification),
      timestamp: new Date().toISOString(),
      color: getTeammateColor(),
    })
    return true
  },
  'Failed to send idle notification to team leader',
  { timeout: 10000 },
)
```

---

## 团队消息类型汇总

| 消息类型 | 方向 | 描述 |
|---------|------|------|
| `permission_request` | 队友 → 队长 | 工具权限请求 |
| `permission_response` | 队长 → 队友 | 权限响应 |
| `sandbox_permission_request` | 队友 → 队长 | 网络权限请求 |
| `plan_approval_request` | 队友 → 队长 | 计划审批请求 |
| `plan_approval_response` | 队长 → 队友 | 计划审批响应 |
| `shutdown_request` | 队长 → 队友 | 关闭请求 |
| `shutdown_approved` | 队友 → 队长 | 关闭确认 |
| `idle_notification` | 队友 → 队长 | 空闲通知 |
| `team_permission_update` | 队长 → 队友 | 团队权限更新 |
| `mode_set_request` | 队长 → 队友 | 权限模式变更 |

---

## 协议流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     团队协议通信流程                                  │
│                                                                     │
│  队友                                                         队长   │
│    │                                                             │   │
│    │ 1. 执行工具，权限不足                                        │   │
│    │    ↓                                                         │   │
│    │ 2. sendPermissionRequestViaMailbox()                        │   │
│    │    → permission_request 消息                                 │   │
│    │    ↓                                                         │   │
│    │                                              ┌───────────────┤   │
│    │                                              │ 3. 队长收到请求│   │
│    │                                              │ 4. 显示确认UI │   │
│    │                                              └───────────────┘   │
│    │    ↓                                                         │   │
│    │ 5. sendPermissionResponseViaMailbox()                       │   │
│    │    ← permission_response 消息                                │   │
│    │                                                             │   │
│    │ 6. 收到响应，继续执行或拒绝                                   │   │
│    │                                                             │   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     计划审批流程                                      │
│                                                                     │
│  队友                                                         队长   │
│    │                                                             │   │
│    │ 1. 计划模式请求                                            │   │
│    │    ↓                                                         │   │
│    │ 2. EnterPlanMode → 阻塞等待审批                             │   │
│    │    → plan_approval_request 消息                             │   │
│    │    ↓                                                         │   │
│    │                                              ┌───────────────┤   │
│    │                                              │ 3. 队长自动审批│   │
│    │                                              └───────────────┘   │
│    │    ↓                                                         │   │
│    │ 4. 收到 plan_approval_response                              │   │
│    │    - approved=true: 过渡到正常模式                          │   │
│    │    - approved=false: 保持计划模式                           │   │
│    │                                                             │   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计

### 1. 消息队列隔离

队长和队友使用独立的邮箱文件，通过文件系统锁实现并发隔离。

### 2. 计划审批验证

```typescript
// 验证消息来自队长，防止伪造
if (approvalResponse && msg.from === 'team-lead') {
  // 只有来自队长的审批才有效
}
```

### 3. 权限模式继承规则

```typescript
// 要求计划模式时不继承绕过权限
if (planModeRequired) {
  // 不继承 bypassPermissions
} else if (permissionMode === 'bypassPermissions') {
  // 继承为 --dangerously-skip-permissions
}
```

### 4. 向后兼容

```typescript
// 没有 agent ID 且有团队上下文时，视为队长
if (!myAgentId) return true
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| sendPermissionRequestViaMailbox | `src/utils/swarm/permissionSync.ts:676-722` |
| sendPermissionResponseViaMailbox | `src/utils/swarm/permissionSync.ts:734-783` |
| getLeaderName | `src/utils/swarm/permissionSync.ts:651-667` |
| isTeamLead | `src/utils/teammate.ts:171-198` |
| PlanApprovalRequestMessageSchema | `src/utils/teammateMailbox.ts:684-697` |
| isStructuredProtocolMessage | `src/utils/teammateMailbox.ts:1073-1095` |
| useInboxPoller 消息路由 | `src/hooks/useInboxPoller.ts:204-248` |
| 队长端权限请求处理 | `src/hooks/useInboxPoller.ts:250-364` |
| 队长自动审批 | `src/hooks/useInboxPoller.ts:599-662` |
| 进程内权限处理 | `src/utils/swarm/inProcessRunner.ts:195-334` |
| 团队允许路径 | `src/utils/swarm/teammateInit.ts:44-79` |
| 空闲通知 | `src/utils/swarm/teammateInit.ts:97-129` |
| getCoordinatorSystemPrompt | `src/coordinator/coordinatorMode.ts:111-369` |
| isCoordinatorMode | `src/hooks/toolPermission/handlers/coordinatorHandler.ts:36-41` |
| buildInheritedCliFlags | `src/tools/shared/spawnMultiAgent.ts:208-260` |

---

*基于 Claude Code v2.1.88 源代码分析*
