# Layer 04: SUB-AGENTS — 子代理

## 概述

**SUB-AGENTS** 是分叉代理（Fork Agent）系统。模型可以创建子任务，在独立上下文中执行。位于 `src/tools/AgentTool/` 和 `src/utils/forkedAgent.ts`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| AgentTool 主实现 | `src/tools/AgentTool/AgentTool.tsx` |
| 分叉子代理逻辑 | `src/tools/AgentTool/forkSubagent.ts` |
| 代理执行 | `src/tools/AgentTool/runAgent.ts` |
| 代理定义加载 | `src/tools/AgentTool/loadAgentsDir.ts` |
| 内置代理 | `src/tools/AgentTool/built-in/` |
| 分叉代理工具 | `src/utils/forkedAgent.ts` |
| 本地代理任务 | `src/tasks/LocalAgentTask/LocalAgentTask.ts` |
| 远程代理任务 | `src/tasks/RemoteAgentTask/RemoteAgentTask.ts` |

---

## 分叉代理 vs 子代理

| 特性 | Fork (分叉) | Subagent (子代理) |
|------|-------------|------------------|
| 上下文 | **独立**，不污染主对话 | 共享部分上下文 |
| 位置 | **后台运行** | 可前台运行 |
| 通信 | 结果可取回 | 实时交互 |
| 用途 | 研究/多步实现 | 并行任务执行 |

---

## 分叉代理架构

### Fork 启用条件

```typescript
// src/tools/AgentTool/forkSubagent.ts:32-39
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // 与协调者模式互斥
    if (getIsNonInteractiveSession()) return false  // 非交互会话禁用
    return true
  }
  return false
}
```

### Fork 代理定义

```typescript
// src/tools/AgentTool/forkSubagent.ts:60-71
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  tools: ['*'],              // 继承父工具池
  maxTurns: 200,
  model: 'inherit',          // 继承父模型
  permissionMode: 'bubble',   // 权限提示冒泡到父终端
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '',
} satisfies BuiltInAgentDefinition
```

### 防止递归分叉

```typescript
// src/tools/AgentTool/forkSubagent.ts:78-89
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

---

## 分叉消息构建

### Prompt Cache 共享策略

```typescript
// src/tools/AgentTool/forkSubagent.ts:107-169
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆完整的 assistant 消息（保留所有 tool_use, thinking, text）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 为所有 tool_use 构建占位符结果
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text' as const, text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 3. 构建最终消息：所有占位符 + 每个子代理的 directive
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text' as const, text: buildChildMessage(directive) },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

### 分叉指令模板

```typescript
// src/tools/AgentTool/forkSubagent.ts:171-198
export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — you ARE the fork. Do NOT spawn sub-agents.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
5. If you modify files, commit your changes before reporting.
6. Do NOT emit text between tool calls. Use tools silently, then report once at the end.
7. Stay strictly within your directive's scope.
8. Keep your report under 500 words.
9. Your response MUST begin with "Scope:".
10. REPORT structured facts, then stop

Output format:
  Scope: <echo back your assigned scope>
  Result: <key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list if any>
</${FORK_BOILERPLATE_TAG}>

${FORK_DIRECTIVE_PREFIX}${directive}`
}
```

---

## runAgent 执行流程

```typescript
// src/tools/AgentTool/runAgent.ts:248-700+
export async function* runAgent({
  agentDefinition,
  promptMessages,
  toolUseContext,
  canUseTool,
  isAsync,
  forkContextMessages,
  querySource,
  override,
  model,
  maxTurns,
  availableTools,
  allowedTools,
  worktreePath,
  // ...
}): AsyncGenerator<Message, void> {
  // 1. 解析代理模型
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model,
    toolUseContext.options.mainLoopModel,
    model,
    permissionMode,
  )

  // 2. 创建代理 ID
  const agentId = override?.agentId ? override.agentId : createAgentId()

  // 3. 构建初始消息
  const contextMessages = forkContextMessages
    ? filterIncompleteToolCalls(forkContextMessages)
    : []
  const initialMessages = [...contextMessages, ...promptMessages]

  // 4. 创建文件状态缓存（隔离）
  const agentReadFileState = forkContextMessages !== undefined
    ? cloneFileStateCache(toolUseContext.readFileState)
    : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)

  // 5. 解析工具
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools

  // 6. 获取系统提示词
  const agentSystemPrompt = override?.systemPrompt
    ? override.systemPrompt
    : asSystemPrompt(await getAgentSystemPrompt(...))

  // 7. 确定中止控制器
  const agentAbortController = override?.abortController
    ? override.abortController
    : isAsync
      ? new AbortController()  // 异步代理独立运行
      : toolUseContext.abortController  // 同步代理共享父控制器

  // 8. 执行 SubagentStart 钩子
  for await (const hookResult of executeSubagentStartHooks(...)) {
    // ...
  }

  // 9. 调用核心 query() 循环
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    toolUseContext: newToolUseContext,
    canUseTool,
    querySource,
    maxTurns,
  })) {
    yield message
  }
}
```

---

## 内置代理类型

### 内置代理清单

| 代理类型 | 文件 | 用途 |
|---------|------|------|
| **Explore** | `built-in/exploreAgent.ts` | 探索代码库 |
| **Plan** | `built-in/planAgent.ts` | 制定计划 |
| **Verification** | `built-in/verificationAgent.ts` | 验证实施 |
| **General Purpose** | `built-in/generalPurposeAgent.ts` | 通用任务 |
| **Claude Code Guide** | `built-in/claudeCodeGuideAgent.ts` | 代码指南 |

### AgentTool 输入模式

```typescript
// src/tools/AgentTool/AgentTool.tsx:82-102
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('Type of specialized agent'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional().describe('Run in background'),
}))

const fullInputSchema = lazySchema(() => {
  return baseInputSchema().merge(z.object({
    name: z.string().optional().describe('Name for spawned agent'),
    team_name: z.string().optional().describe('Team name'),
    mode: permissionModeSchema().optional().describe('Permission mode'),
    isolation: z.enum(['worktree']).optional().describe('Isolation mode'),
    cwd: z.string().optional().describe('Working directory override'),
  }))
})
```

---

## 隔离模式

### Worktree 隔离

```typescript
// src/tools/AgentTool/AgentTool.tsx:42
import { createAgentWorktree, hasWorktreeChanges, removeAgentWorktree } from '../../utils/worktree.js'

// 创建工作区隔离的代理
if (input.isolation === 'worktree') {
  const worktreePath = await createAgentWorktree(agentId, cwd)
  // 代理在独立的 git worktree 中运行
}
```

---

## 异步 vs 同步代理

### 异步代理

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.ts
export async function createAsyncAgent(params: {
  agentDefinition: AgentDefinition
  prompt: string
  description: string
}): Promise<{ agentId: string; outputFile: string }> {
  // 创建异步代理任务
  // 后台运行，结果写入文件
  // 完成后通过 notifyCommandLifecycle 通知
}
```

### 同步代理

```typescript
// 直接在当前循环中执行
for await (const message of runAgent({ ... })) {
  yield message  // 消息直接产出到父循环
}
```

---

## 代理上下文隔离

### createSubagentContext

```typescript
// src/utils/forkedAgent.ts
export function createSubagentContext(
  parentContext: ToolUseContext,
  agentId: AgentId,
): ToolUseContext {
  return {
    ...parentContext,
    agentId,
    readFileState: cloneFileStateCache(parentContext.readFileState),
    abortController: new AbortController(),
    messages: [],  // 清空消息历史
  }
}
```

### 权限模式处理

```typescript
// runAgent.ts:416-498
const agentGetAppState = () => {
  let toolPermissionContext = state.toolPermissionContext

  // 代理可以覆盖权限模式
  if (agentPermissionMode && parentMode === 'default') {
    toolPermissionContext = {
      ...toolPermissionContext,
      mode: agentPermissionMode,
    }
  }

  // 后台代理应避免权限提示
  if (isAsync && !canShowPermissionPrompts) {
    toolPermissionContext = {
      ...toolPermissionContext,
      shouldAvoidPermissionPrompts: true,
    }
  }

  return { ...state, toolPermissionContext }
}
```

---

## 对话流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     分叉代理工作流程                                 │
│                                                                     │
│  主代理                                                            │
│    │                                                               │
│    │ AgentTool(directive="研究 src/auth 模块")                      │
│    ↓                                                               │
│  buildForkedMessages(directive, assistantMessage)                   │
│    │                                                               │
│    │ 1. 克隆完整的 assistant 消息                                   │
│    │ 2. 为所有 tool_use 构建占位符                                 │
│    │ 3. 添加 fork directive                                        │
│    ↓                                                               │
│  createSubagentContext(parentContext, agentId)                      │
│    │                                                               │
│    │ - agentId: 新 ID                                              │
│    │ - readFileState: 克隆                                         │
│    │ - messages: []                                               │
│    ↓                                                               │
│  runAgent(forkAgent, messages, newContext)                          │
│    │                                                               │
│    │ query() 循环开始                                              │
│    ↓                                                               │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ 分叉代理独立执行                                               │ │
│  │                                                               │ │
│  │ 1. 使用独立的消息历史                                         │ │
│  │ 2. 可以读写文件（不污染父对话）                               │ │
│  │ 3. 完成后提交结果                                             │ │
│  │                                                               │ │
│  └───────────────────────────────────────────────────────────────┘ │
│    │                                                               │
│    │ fork 结果注入父对话                                          │
│    ↓                                                               │
│  主代理继续处理                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 与其他层的关系

```
Layer 01: THE LOOP
    │
    └── Layer 02: TOOL DISPATCH
            │
            └── Layer 04: SUB-AGENTS
                    │
                    ├── 创建新的 query 调用
                    │
                    └── 与 Layer 03 (PLANNING) 协作
                            │
                            └── PlanAgent 用于计划模式
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| isForkSubagentEnabled | `src/tools/AgentTool/forkSubagent.ts:32-39` |
| FORK_AGENT 定义 | `src/tools/AgentTool/forkSubagent.ts:60-71` |
| buildForkedMessages | `src/tools/AgentTool/forkSubagent.ts:107-169` |
| buildChildMessage | `src/tools/AgentTool/forkSubagent.ts:171-198` |
| isInForkChild | `src/tools/AgentTool/forkSubagent.ts:78-89` |
| runAgent | `src/tools/AgentTool/runAgent.ts:248-700+` |
| createSubagentContext | `src/utils/forkedAgent.ts:54-140` |
| AgentTool 主实现 | `src/tools/AgentTool/AgentTool.tsx:196+` |
| 异步代理创建 | `src/tasks/LocalAgentTask/LocalAgentTask.ts` |

---

*基于 Claude Code v2.1.88 源代码分析*
