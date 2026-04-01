# Layer 01: THE LOOP — 核心循环

## 概述

**THE LOOP** 是 Claude Code Harness 的心脏，位于 `src/query.ts`。它是一个 `while(true)` 的 async generator 模式，持续执行 fetch-execute 循环，直到任务完成或满足退出条件。

---

## 源码位置

| 项目 | 路径 |
|------|------|
| 主文件 | `src/query.ts` |
| 相关配置 | `src/query/config.ts` |
| 依赖注入 | `src/query/deps.ts` |
| 停止钩子 | `src/query/stopHooks.ts` |
| Token预算 | `src/query/tokenBudget.ts` |

---

## 核心架构

### 1. 导出函数 vs 内部循环

```typescript
// src/query.ts:219-239
// 导出函数：包装 queryLoop，添加命令生命周期通知
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // 仅在 queryLoop 正常返回时到达。throw 时跳过。
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}

// src/query.ts:241-1729
// 内部循环：包含 while(true) 的核心逻辑
async function* queryLoop(params, consumedCommandUuids): AsyncGenerator<...> {
  // ... 初始化代码 ...
  while (true) {
    // 5个阶段
  }
}
```

**关键设计**：`query()` 是对外的 async generator 接口，`queryLoop()` 包含实际的 `while(true)` 循环。这种分离允许：
- 对外接口干净，只处理命令生命周期
- 内部循环可以有更复杂的控制流

---

## 状态管理

### State 结构体

```typescript
// src/query.ts:204-217
// 跨迭代可变的状态，在每次循环顶部解构
type State = {
  messages: Message[]                                    // 对话历史
  toolUseContext: ToolUseContext                          // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined // 压缩跟踪状态
  maxOutputTokensRecoveryCount: number                    // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean                    // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined              // 输出 token 上限覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined // 待处理的工具摘要
  stopHookActive: boolean | undefined                    // 停止钩子是否激活
  turnCount: number                                      // 轮次计数器
  transition: Continue | undefined                       // 上一轮继续原因
}
```

**设计意图**：所有可变状态集中在 `State` 结构体中，循环顶部统一解构。写入时使用 `state = { ... }` 而非单独赋值，保证状态一致性。

### 初始化

```typescript
// src/query.ts:268-291
let state: State = {
  messages: params.messages,
  toolUseContext: params.toolUseContext,
  maxOutputTokensOverride: params.maxOutputTokensOverride,
  autoCompactTracking: undefined,
  stopHookActive: undefined,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  turnCount: 1,
  pendingToolUseSummary: undefined,
  transition: undefined,
}
```

---

## 循环五阶段

```
┌─────────────────────────────────────────────────────────────────────┐
│                        while (true) {                              │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Phase 1: SETUP PHASE (line 560-614)                         │  │
│  │ - 初始化 StreamingToolExecutor                              │  │
│  │ - 检查 token 警告状态                                        │  │
│  │ - 检查是否达到阻塞限制                                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Phase 2: API CALL PHASE (line 652-954)                       │  │
│  │ - 调用 deps.callModel() 流式 API                            │  │
│  │ - 处理 streaming fallback                                    │  │
│  │ - 提取 tool_use 块                                          │  │
│  │ - 边接收边执行工具 (streamingToolExecutor)                   │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Phase 3: RECOVERY PHASE (line 1062-1256)                     │  │
│  │ - 处理 prompt-too-long 错误                                 │  │
│  │ - 处理 max-output-tokens 错误                               │  │
│  │ - 处理 media-size 错误                                      │  │
│  │ - 执行 context collapse drain                               │  │
│  │ - 执行 reactive compact                                     │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Phase 4: TOOL EXECUTION PHASE (line 1360-1521)               │  │
│  │ - 消费 streamingToolExecutor 的剩余结果                      │  │
│  │ - 或调用 runTools() 执行工具                                │  │
│  │ - 生成工具使用摘要                                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              ↓                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Phase 5: CONTINUATION PHASE (line 1599-1728)                 │  │
│  │ - 处理附件消息                                               │  │
│  │ - 处理记忆预取                                               │  │
│  │ - 刷新工具定义                                               │  │
│  │ - 检查 maxTurns 限制                                        │  │
│  │ - 递归调用: state = { ...messages, turnCount++ }            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: SETUP PHASE

### 源码位置

`src/query.ts:560-614`

### 关键代码

```typescript
// src/query.ts:560-568
queryCheckpoint('query_setup_start')
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null

// src/query.ts:570-578
const appState = toolUseContext.getAppState()
const permissionMode = appState.toolPermissionContext.mode
let currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens:
    permissionMode === 'plan' &&
    doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

### 功能

1. **创建 StreamingToolExecutor**：如果启用流式工具执行，创建一个实例来管理工具执行
2. **确定运行时模型**：根据权限模式和上下文选择合适的模型
3. **检查阻塞限制**：在 API 调用前检查是否达到 token 上限

---

## Phase 2: API CALL PHASE

### 源码位置

`src/query.ts:652-954`

### 核心调用

```typescript
// src/query.ts:659-708
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  // ... 更多选项
})) {
  // 处理每个流式消息
}
```

### 消息处理流程

```typescript
// src/query.ts:826-862
if (message.type === 'assistant') {
  assistantMessages.push(message)

  const msgToolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true
  }

  // 边接收边执行工具
  if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)
    }
  }
}

// 获取已完成工具的结果
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message
      toolResults.push(...normalizeMessagesForAPI([result.message], ...))
    }
  }
}
```

### 关键设计

1. **边接收边执行**：当 `streamingToolExecution` 启用时，工具在模型返回时立即执行，不必等待完整响应
2. **tool_use 块收集**：`needsFollowUp` 标志指示是否需要执行工具
3. **streaming fallback 处理**：如果模型回退发生，生成 tombstone 消息清除孤立的部分消息

---

## Phase 3: RECOVERY PHASE

### 源码位置

`src/query.ts:1062-1256`

### 错误类型和处理

| 错误类型 | 处理方式 | 源码位置 |
|---------|---------|----------|
| **prompt-too-long (413)** | 1. 先尝试 context collapse drain<br>2. 再尝试 reactive compact<br>3. 最后 surface 错误 | line 1085-1183 |
| **max-output-tokens** | 1. 先升级到 64k<br>2. 再尝试恢复消息<br>3. 最多重试 3 次 | line 1188-1256 |
| **media-size 错误** | 使用 reactive compact 剥离媒体重试 | line 1082-1084, 1119-1166 |

### prompt-too-long 恢复流程

```typescript
// src/query.ts:1085-1117
if (isWithheld413) {
  // First: drain all staged context-collapses
  if (feature('CONTEXT_COLLAPSE') && contextCollapse &&
      state.transition?.reason !== 'collapse_drain_retry') {
    const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
    if (drained.committed > 0) {
      state = {
        messages: drained.messages,
        transition: { reason: 'collapse_drain_retry', committed: drained.committed },
        // ...
      }
      continue  // 重试
    }
  }
}

// Second: try reactive compact
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({...})
  if (compacted) {
    // 构建压缩后的消息，继续
    continue
  }
}
```

### max-output-tokens 恢复流程

```typescript
// src/query.ts:1188-1221
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 第一次：升级到 64k
  if (capEnabled && maxOutputTokensOverride === undefined) {
    state = {
      maxOutputTokensOverride: ESCALATED_MAX_TOKENS, // 64k
      transition: { reason: 'max_output_tokens_escalate' },
    }
    continue
  }

  // 后续：添加恢复消息重试
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly — no apology, no recap.`,
      isMeta: true,
    })
    state = {
      messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
      maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
      transition: { reason: 'max_output_tokens_recovery', attempt: ... },
    }
    continue
  }
}
```

---

## Phase 4: TOOL EXECUTION PHASE

### 源码位置

`src/query.ts:1360-1521`

### 两种执行模式

```typescript
// src/query.ts:1380-1382
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()  // 流式：消费剩余结果
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)  // 阻塞式
```

### 执行流程

```typescript
// src/query.ts:1384-1408
for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message  // 产出工具结果

    // 检查钩子是否阻止继续
    if (update.message.type === 'attachment' &&
        update.message.attachment.type === 'hook_stopped_continuation') {
      shouldPreventContinuation = true
    }

    toolResults.push(
      ...normalizeMessagesForAPI([update.message], toolUseContext.options.tools)
        .filter(_ => _.type === 'user')
    )
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

### 工具使用摘要

```typescript
// src/query.ts:1411-1482
// 在工具批次完成后生成摘要，传递给下一轮
if (config.gates.emitToolUseSummaries && toolUseBlocks.length > 0) {
  nextPendingToolUseSummary = generateToolUseSummary({
    tools: toolInfoForSummary,
    signal: toolUseContext.abortController.signal,
    lastAssistantText,
  }).then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
}
```

---

## Phase 5: CONTINUATION PHASE

### 源码位置

`src/query.ts:1599-1728`

### 附件处理

```typescript
// src/query.ts:1580-1590
for await (const attachment of getAttachmentMessages(
  null,
  updatedToolUseContext,
  null,
  queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults],
  querySource,
)) {
  yield attachment
  toolResults.push(attachment)
}
```

### 记忆预取消费

```typescript
// src/query.ts:1599-1614
if (pendingMemoryPrefetch?.settledAt !== null &&
    pendingMemoryPrefetch.consumedOnIteration === -1) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise,
    toolUseContext.readFileState,
  )
  for (const memAttachment of memoryAttachments) {
    const msg = createAttachmentMessage(memAttachment)
    yield msg
    toolResults.push(msg)
  }
  pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
}
```

### 递归调用

```typescript
// src/query.ts:1715-1727
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
state = next
// 继续下一轮 while(true)
```

---

## 退出条件 (Terminal)

### 退出原因类型

| reason | 含义 | 触发位置 |
|--------|------|----------|
| `'completed'` | 任务正常完成 | line 1357 |
| `'aborted_streaming'` | API 流被中止 | line 1051 |
| `'aborted_tools'` | 工具执行被中止 | line 1515 |
| `'prompt_too_long'` | 上下文超出限制且恢复失败 | line 1175 |
| `'image_error'` | 图片/媒体大小错误 | line 1175, 977 |
| `'blocking_limit'` | 达到硬性阻塞限制 | line 646 |
| `'model_error'` | 模型运行时错误 | line 996 |
| `'stop_hook_prevented'` | 停止钩子阻止继续 | line 1279 |
| `'max_turns'` | 达到最大轮次限制 | line 1711 |
| `'hook_stopped'` | 钩子停止了继续 | line 1520 |

---

## 继续原因 (Continue.transition)

### transition 类型

| reason | 含义 | 触发位置 |
|--------|------|----------|
| `'next_turn'` | 正常下一轮 | line 1725 |
| `'stop_hook_blocking'` | 停止钩子阻塞 | line 1302 |
| `'max_output_tokens_escalate'` | 输出 token 上限升级 | line 1217 |
| `'max_output_tokens_recovery'` | 输出 token 恢复重试 | line 1245 |
| `'collapse_drain_retry'` | 上下文压缩排出重试 | line 1110 |
| `'reactive_compact_retry'` | 响应式压缩重试 | line 1162 |
| `'token_budget_continuation'` | Token 预算继续 | line 1338 |

---

## 依赖注入 (QueryDeps)

```typescript
// src/query/deps.ts:21-31
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming      // 模型调用
  microcompact: typeof microcompactMessages       // 微压缩
  autocompact: typeof autoCompactIfNeeded        // 自动压缩
  uuid: () => string                             // UUID 生成
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

**设计意图**：将 I/O 依赖注入，允许测试注入 fake 而非 spyOn 模块。

---

## 查询配置 (QueryConfig)

```typescript
// src/query/config.ts:15-27
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean  // 流式工具执行
    emitToolUseSummaries: boolean    // 发出工具摘要
    isAnt: boolean                    // 是否为 Anthropic 内部用户
    fastModeEnabled: boolean          // 快速模式
  }
}
```

---

## 关键设计原则

### 1. AsyncGenerator 模式

`query()` 和 `queryLoop()` 都是 async generator，通过 `yield*` 链接。这允许：
- 流式输出：消息可以边生成边 yield 给调用者
- 双向通信：外部可以通过 generator 的 throw 方法注入事件

### 2. 状态一致性

使用 `state = { ... }` 原子性更新状态，而非分别赋值。这确保了：
- 如果循环在更新中间被中断，状态不会处于部分更新状态
- 更容易推理状态转换

### 3. 延迟计算

`messagesForQuery` 在每次迭代顶部从 `state.messages` 复制，确保：
- 压缩等操作在副本上进行
- 原始 `state.messages` 保留完整历史用于调试

### 4. 特征门控

所有 `feature()` 调用都在条件语句内（if/ternary），满足 bun:bundle 的 tree-shaking 要求。

---

## 与其他层的关系

```
Layer 01: THE LOOP
    │
    ├── Layer 02: TOOL DISPATCH (tools 来自 toolUseContext.options.tools)
    │       │
    │       ├── Layer 03: PLANNING (ExitPlanModeTool 控制计划模式)
    │       │
    │       ├── Layer 04: SUB-AGENTS (创建新的 query 调用)
    │       │
    │       └── Layer 05: KNOWLEDGE ON DEMAND (通过 getAttachmentMessages 注入)
    │
    └── Layer 06: CONTEXT COMPRESSION (通过 deps.autocompact 调用)
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| query() 导出函数 | `src/query.ts:219-239` |
| queryLoop() 内部循环 | `src/query.ts:241-1729` |
| State 类型定义 | `src/query.ts:204-217` |
| while(true) 循环开始 | `src/query.ts:307` |
| Setup Phase | `src/query.ts:560-614` |
| API Call Phase | `src/query.ts:652-954` |
| Streaming tool execution | `src/query.ts:837-844` |
| Recovery Phase | `src/query.ts:1062-1256` |
| Tool Execution Phase | `src/query.ts:1360-1521` |
| Continuation Phase | `src/query.ts:1599-1728` |
| 递归调用 | `src/query.ts:1715-1727` |
| QueryDeps 定义 | `src/query/deps.ts:21-31` |
| QueryConfig 定义 | `src/query/config.ts:15-27` |
| 生产依赖工厂 | `src/query/deps.ts:33-40` |

---

*基于 Claude Code v2.1.88 源代码分析*
