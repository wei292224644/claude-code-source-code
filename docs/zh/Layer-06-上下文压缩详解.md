# Layer 06: CONTEXT COMPRESSION — 上下文压缩

## 概述

**CONTEXT COMPRESSION** 是三层压缩系统，防止上下文溢出。位于 `src/services/compact/`。

注意：由于 108 个特性门控模块在发布版本中被消除，部分压缩模块（如 `snipCompact.ts`、`contextCollapse`）在源代码中不可见，但压缩逻辑的核心接口仍然存在。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 压缩核心 | `src/services/compact/compact.ts` |
| 自动压缩 | `src/services/compact/autoCompact.ts` |
| 微压缩 | `src/services/compact/microCompact.ts` |
| 压缩提示词 | `src/services/compact/prompt.ts` |
| 响应式压缩 | `src/services/compact/reactiveCompact.ts` |
| 上下文分析 | `src/utils/contextAnalysis.ts` |

---

## 三层压缩策略

| 层级 | 名称 | Feature Flag | 描述 |
|------|------|-------------|------|
| **Layer 1** | snipCompact | `HISTORY_SNIP` | 裁剪历史**中间**消息 |
| **Layer 2** | microcompact | `CACHED_MICROCOMPACT` | 单轮内轻量压缩 |
| **Layer 3** | autoCompact | (始终启用) | 上下文接近上限时**总结** |

---

## Layer 1: snipCompact (历史裁剪)

### 功能

裁剪历史消息的中间部分，保留头尾：

```
上下文增长:
[msg1][msg2][msg3]...[msgN][recent]
    ↓
snipCompact: 删除中间部分，保留头尾
[msg1]...[recent]
```

### 源码位置

```typescript
// src/query.ts:401-409
if (feature('HISTORY_SNIP')) {
  queryCheckpoint('query_snip_start')
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
  queryCheckpoint('query_snip_end')
}
```

### snipTokensFreed 传递给 autoCompact

```typescript
// src/query.ts:397-410
// Apply snip before microcompact (both may run — they are not mutually exclusive).
// snipTokensFreed is plumbed to autocompact so its threshold check reflects
// what snip removed; tokenCountWithEstimation alone can't see it.
let snipTokensFreed = 0
if (feature('HISTORY_SNIP')) {
  // ...
  snipTokensFreed = snipResult.tokensFreed
}
```

---

## Layer 2: microcompact (微压缩)

### 功能

在单轮内进行轻量压缩，主要用于工具结果缓存。

### 源码位置

```typescript
// src/query.ts:412-426
queryCheckpoint('query_microcompact_start')
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages

// 对于缓存的 microcompact (cache editing)，延迟 boundary message 直到 API 响应后
const pendingCacheEdits = feature('CACHED_MICROCOMPACT')
  ? microcompactResult.compactionInfo?.pendingCacheEdits
  : undefined
queryCheckpoint('query_microcompact_end')
```

### microcompactMessages

```typescript
// src/services/compact/microCompact.ts
export async function microcompactMessages(
  messages: Message[],
  toolUseContext: ToolUseContext,
  querySource: string,
): Promise<{ messages: Message[], compactionInfo?: any }> {
  // 1. 工具结果缓存
  // 2. 清理重复内容
  // ...
}
```

---

## Layer 3: autoCompact (自动压缩)

### 功能

当上下文接近 token 限制时进行总结。

### 阈值计算

```typescript
// src/services/compact/autoCompact.ts:72-91
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  // 13,000 token 缓冲
  return autocompactThreshold
}

// 预留 20,000 token 用于压缩输出
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
```

### 压缩触发检查

```typescript
// src/services/compact/autoCompact.ts:93-145
export function calculateTokenWarningState(
  tokenUsage: number,
  model: string,
): {
  percentLeft: number
  isAboveWarningThreshold: boolean
  isAboveErrorThreshold: boolean
  isAboveAutoCompactThreshold: boolean
  isAtBlockingLimit: boolean
} {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  // ...
  const isAboveAutoCompactThreshold =
    isAutoCompactEnabled() && tokenUsage >= autoCompactThreshold
  // ...
}
```

### autoCompactIfNeeded

```typescript
// src/services/compact/autoCompact.ts:241-351
export async function autoCompactIfNeeded(
  messages: Message[],
  toolUseContext: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  querySource?: string,
  tracking?: AutoCompactTrackingState,
  snipTokensFreed?: number,
): Promise<{
  wasCompacted: boolean
  compactionResult?: CompactionResult
  consecutiveFailures?: number
}> {
  // 1. 电路断路器：连续失败 N 次后停止
  if (tracking?.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }
  }

  // 2. 检查是否应该压缩
  const shouldCompact = await shouldAutoCompact(messages, model, ...)
  if (!shouldCompact) {
    return { wasCompacted: false }
  }

  // 3. 尝试会话记忆压缩
  const sessionMemoryResult = await trySessionMemoryCompaction(...)
  if (sessionMemoryResult) {
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 4. 执行标准压缩
  try {
    const compactionResult = await compactConversation(
      messages,
      toolUseContext,
      cacheSafeParams,
      true, // 抑制用户问题
      undefined, // 无自定义指令
      true, // isAutoCompact
      recompactionInfo,
    )
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    // 记录连续失败
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

---

## compactConversation - 压缩核心

```typescript
// src/services/compact/compact.ts
export async function compactConversation(
  messages: Message[],
  toolUseContext: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressUserQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean,
  recompactionInfo: RecompactionInfo,
): Promise<CompactionResult> {
  // 1. 剥离图像（图像对生成摘要无用）
  const messagesForCompact = stripImagesFromMessages(messages)

  // 2. 调用压缩模型生成摘要
  const summary = await queryCompactModel(
    messagesForCompact,
    cacheSafeParams,
    customInstructions,
  )

  // 3. 构建压缩后的消息
  const summaryMessages = buildCompactMessages(summary, messagesForCompact)

  // 4. 处理压缩后的清理
  runPostCompactCleanup(querySource)

  return {
    summaryMessages,
    preCompactTokenCount,
    postCompactTokenCount,
  }
}
```

---

## 响应式压缩 (reactiveCompact)

### 功能

当 API 返回 413 错误时，尝试恢复。

```typescript
// src/services/compact/reactiveCompact.ts
export async function tryReactiveCompact(params: {
  hasAttempted: boolean
  querySource: string
  aborted: boolean
  messages: Message[]
  cacheSafeParams: CacheSafeParams
}): Promise<CompactionResult | null> {
  if (params.hasAttempted) {
    return null  // 已经尝试过
  }

  // 尝试剥离媒体并重试
  // ...
}
```

---

## 压缩边界消息

```typescript
// src/utils/messages.ts
export function createCompactBoundaryMessage(
  type: 'snip' | 'microcompact' | 'autocompact',
  tokensRemoved: number,
): SystemCompactBoundaryMessage {
  return {
    type: 'system',
    subtype: 'compact_boundary',
    boundaryType: type,
    tokensRemoved,
  }
}
```

---

## 查询循环中的压缩集成

```typescript
// src/query.ts:340-470
while (true) {
  // 1. Apply snip (Layer 1)
  if (feature('HISTORY_SNIP')) {
    const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
    messagesForQuery = snipResult.messages
    snipTokensFreed = snipResult.tokensFreed
  }

  // 2. Apply microcompact (Layer 2)
  const microcompactResult = await deps.microcompact(messagesForQuery, ...)

  // 3. Apply context collapse (feature-gated)
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
    messagesForQuery = collapseResult.messages
  }

  // 4. Check autocompact (Layer 3)
  const { compactionResult, consecutiveFailures } = await deps.autocompact(
    messagesForQuery,
    toolUseContext,
    { systemPrompt, userContext, ... },
    querySource,
    tracking,
    snipTokensFreed,
  )

  if (compactionResult) {
    // 压缩成功，产出边界消息
    const postCompactMessages = buildPostCompactMessages(compactionResult)
    for (const message of postCompactMessages) {
      yield message
    }
    messagesForQuery = postCompactMessages
  }
}
```

---

## 压缩结果构建

```typescript
// src/services/compact/compact.ts
export function buildPostCompactMessages(
  compactionResult: CompactionResult
): Message[] {
  const { summaryMessages, attachments, hookResults } = compactionResult

  return [
    // 压缩摘要消息
    createCompactBoundaryMessage('autocompact', tokensRemoved),
    ...summaryMessages,
    // 附件（工具结果等）
    ...attachments,
    // 钩子结果
    ...hookResults,
  ]
}
```

---

## 关键设计

### 1. 电路断路器

```typescript
// 连续失败 N 次后停止尝试压缩
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### 2. 压缩层级顺序

```
snipCompact (历史裁剪)
    ↓
microcompact (微压缩)
    ↓
contextCollapse (上下文折叠)
    ↓
autoCompact (自动压缩总结)
```

### 3. 图像剥离

```typescript
// stripImagesFromMessages - 图像对摘要无用，且可能导致压缩 API 调用超时
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    if (message.type !== 'user') {
      return message
    }
    // 替换图像块为文本标记
    // ...
  })
}
```

### 4. snipTokensFreed 传递

```typescript
// snip 释放的 token 反映到 autoCompact 阈值检查
// 否则会用 stale 的 token 计数
```

---

## 流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        压缩流程                                       │
│                                                                     │
│  消息增长 → [msg1][msg2][msg3]...[msgN][recent]                    │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 1: snipCompact (HISTORY_SNIP)                          │   │
│  │ 删除中间历史消息，保留头尾                                    │   │
│  │ tokensFreed = X                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 2: microcompact (CACHED_MICROCOMPACT)                   │   │
│  │ 工具结果缓存，轻量压缩                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 3: contextCollapse (CONTEXT_COLLAPSE)                   │   │
│  │ 折叠累积的上下文视图                                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 4: autoCompact                                          │   │
│  │ 检查阈值 (contextWindow - 13000)                              │   │
│  │ 如果超过阈值，调用 compactConversation()                      │   │
│  │                                                              │   │
│  │ 1. 剥离图像                                                 │   │
│  │ 2. 调用压缩模型                                             │   │
│  │ 3. 构建摘要消息                                             │   │
│  │ 4. 替换原始消息                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  [summary][recent messages] → 继续循环                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| autoCompactIfNeeded | `src/services/compact/autoCompact.ts:241-351` |
| calculateTokenWarningState | `src/services/compact/autoCompact.ts:93-145` |
| getAutoCompactThreshold | `src/services/compact/autoCompact.ts:72-91` |
| compactConversation | `src/services/compact/compact.ts` |
| stripImagesFromMessages | `src/services/compact/compact.ts:145-200` |
| microcompactMessages | `src/services/compact/microCompact.ts` |
| tryReactiveCompact | `src/services/compact/reactiveCompact.ts` |
| 压缩集成 | `src/query.ts:340-470` |
| snipCompact 集成 | `src/query.ts:401-409` |

---

*基于 Claude Code v2.1.88 源代码分析*
