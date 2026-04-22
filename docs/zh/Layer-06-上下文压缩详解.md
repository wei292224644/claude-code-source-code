# Layer 06: CONTEXT COMPRESSION — 上下文压缩详解

> 基于 Claude Code v2.1.88 源代码分析

## 概述

Claude Code 的上下文压缩是一个**四层过滤系统**，在每轮查询循环中按顺序执行，目的是在 token 使用量接近上下文窗口上限时，通过逐层压缩避免触发 API 的 `prompt_too_long` 错误。整个压缩系统在 `src/services/compact/` 和 `src/query.ts` 中实现。

### 四层压缩策略

| 层级 | 名称 | Feature Flag | 描述 |
|------|------|-------------|------|
| **Layer 1** | snipCompact | `HISTORY_SNIP` | 裁剪历史**中间**消息，保留头尾 |
| **Layer 2** | microcompact | `CACHED_MICROCOMPACT` | 单轮内轻量压缩，处理工具结果 |
| **Layer 3** | contextCollapse | `CONTEXT_COLLAPSE` | 折叠累积的上下文视图 |
| **Layer 4** | autoCompact | (始终启用) | 上下文接近上限时**总结** |

---

## 自然语言流程说明

当你与 Claude Code 对话时，你的对话历史（包括你发送的消息、Claude 的回复、工具调用结果等）会随着对话进行而不断增长。这些历史消息需要作为上下文发送给 API，而 API 的上下文窗口是有限的。当上下文增长到接近上限时，Claude Code 就需要开始"减肥"——这就是上下文压缩系统的工作。

### 一、为什么需要压缩？

API 的上下文窗口有上限（比如 200K tokens），但 Claude Code 需要留出空间给系统提示词、工具定义、用户上下文，以及压缩后的输出。实际可用的上下文大约是窗口大小减去 13,000 tokens 的 buffer。当可用的 token 快用完时，Claude Code 必须在上下文"爆掉"之前主动压缩，否则 API 会返回 `prompt_too_long` 错误，用户就卡住了。

### 二、四层压缩如何配合工作？

Claude Code 采用了"先轻量后重量"的策略——先用开销低的方式清理一些空间，如果还不够，再逐步升级到更彻底但也更昂贵的压缩方式。

**第一层：snipCompact（历史裁剪）** — 这是最简单的一刀。它看到对话历史里有很多消息，其中早期和最近的消息最重要（早期有初始指令，最近有当前任务上下文），而中间的消息往往是一些中间步骤、调试信息，对当前任务贡献最小。snipCompact 把中间部分直接删掉，保留头尾，让上下文"缩短"但保留关键信息。

**第二层：microcompact（微压缩）** — 这一层专门处理工具结果。在一次对话中，Claude 可能调用了 Read 读取文件、Grep 搜索代码、Bash 执行命令等几十次。每次工具调用都会产生结果，这些结果占据大量 tokens，但很多已经"用过"了。microcompact 有两种策略：一是时间触发——如果用户很久没说话了（比如 30 分钟），服务器端的缓存会过期，此时提前把旧工具结果清理掉可以减少缓存重写的开销；二是数量触发——当工具结果积累到一定数量时，通过 API 的 `cache_edits` 机制删除旧的结果，但不改变本地的消息内容，这样服务器端缓存的 prefix 仍然有效。

**第三层：contextCollapse（上下文折叠）** — 这是更细粒度的上下文管理，把多个消息"折叠"成更少的视图。这层启用时，autoCompact 会被抑制，因为两者会竞争。

**第四层：autoCompact（自动压缩总结）** — 这是最后一层，也是最彻底的一层。当前三层都运行之后，如果 token 计数仍然超过阈值，autoCompact 就会启动。它把所有历史消息发送给 API，让 API 用另一个模型调用生成一段摘要。这段摘要"压缩"了之前所有对话的要点，然后原始的历史消息被替换成这段摘要，从而大幅缩小上下文体积。

### 三、压缩之后会发生什么？

压缩不是简单地删除内容就结束了。压缩后，系统会：

- 输出一条边界消息告诉用户"之前的对话已经被压缩了"
- 清空文件读取缓存（因为压缩后的上下文不包含之前的文件读取结果）
- 重新注入用户最近访问的文件内容（最多 5 个，每个最多 5,000 tokens）
- 重新注入用户使用的技能（Skill）内容（每个最多 5,000 tokens，总共最多 25,000 tokens）
- 重新注入当前计划文件内容（如果有）
- 重新注入工具、代理、MCP 的状态信息（让模型知道当前有哪些工具可用）
- 执行 SessionStart hooks（压缩本质上是一个新的"会话开始"）

### 四、关键设计细节

**电路断路器**：如果压缩连续失败 3 次（比如上下文已经无法压缩到足够小），系统会停止尝试，避免浪费 API 调用。

**PTL 重试**：如果压缩请求本身触发了 `prompt_too_long`（压缩需要太多 token），系统会尝试删除最老的部分对话历史，然后重试，最多 3 次。

**snipTokensFreed 传递**：snipCompact 删除消息后，autoCompact 的阈值检查需要知道"删掉了多少 tokens"，否则它会用 snip 之前过大的 token 计数来判断，导致误判。

**图像剥离**：压缩时，所有图片和文档内容都会被替换成 `[image]` 或 `[document]` 文本标记，因为图片对生成摘要没有帮助，而且可能导致压缩请求本身超时。

**缓存提示词共享**：压缩时，系统会尝试复用主对话已经缓存好的提示词 prefix，避免重新创建缓存。测试显示，这可以让 98% 的压缩请求命中缓存。

### 五、发布版本中被消除的模块

`sni pCompact`、`contextCollapse`、部分 `microcompact` 功能使用了 Bun 的编译时 `feature()` intrinsic。在 Anthropic 内部构建中，这些 feature 返回 `true`，但发布到 npm 的外部版本使用了永远返回 `false` 的 stub，导致相关模块的代码被消除。具体来说：

| 特性 | 内部版本 | npm 发布版本 |
|------|---------|-------------|
| `HISTORY_SNIP` | 启用 | 禁用（snipCompact 不可用） |
| `CONTEXT_COLLAPSE` | 启用 | 禁用（contextCollapse 不可用） |
| `CACHED_MICROCOMPACT` | 启用 | 禁用（缓存微压缩不可用） |

这是 Anthropic 分阶段发布策略的一部分——新特性先在内部测试，稳定后再逐步向外部用户开放。

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
| 缓存微压缩 | `src/services/compact/cachedMicrocompact.js` |
| 会话记忆压缩 | `src/services/compact/sessionMemoryCompact.ts` |

---

## 查询循环中的压缩集成

在 `src/query.ts` 的主循环中，压缩按以下顺序执行：

```typescript
// src/query.ts:396-470
while (true) {
  // === 1. snipCompact (Layer 1) ===
  if (feature('HISTORY_SNIP')) {
    queryCheckpoint('query_snip_start')
    const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
    messagesForQuery = snipResult.messages
    snipTokensFreed = snipResult.tokensFreed  // 传递给 autoCompact
    if (snipResult.boundaryMessage) {
      yield snipResult.boundaryMessage
    }
    queryCheckpoint('query_snip_end')
  }

  // === 2. microcompact (Layer 2) ===
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

  // === 3. contextCollapse (Layer 3, feature-gated) ===
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
    messagesForQuery = collapseResult.messages
  }

  // === 4. autoCompact (Layer 4) ===
  queryCheckpoint('query_autocompact_start')
  const { compactionResult, consecutiveFailures } = await deps.autocompact(
    messagesForQuery,
    toolUseContext,
    {
      systemPrompt,
      userContext,
      systemContext,
      toolUseContext,
      forkContextMessages: messagesForQuery,
    },
    querySource,
    tracking,
    snipTokensFreed,  // 使用 snip 之后的 token 计数
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

## Layer 1: snipCompact（历史消息裁剪）

### 功能

裁剪历史消息的**中间部分**，保留头部的系统提示和尾部的最近消息。当对话历史很长时，中间的消息对当前任务贡献最小但消耗最多 token，因此被删除。

### 上下文增长示意

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

### 关键设计：snipTokensFreed 传递

```typescript
// src/query.ts:396-410
// Apply snip before microcompact (both may run — they are not mutually exclusive).
// snipTokensFreed is plumbed to autocompact so its threshold check reflects
// what snip removed; tokenCountWithEstimation alone can't see it (reads usage
// from the protected-tail assistant, which survives snip unchanged).
let snipTokensFreed = 0
if (feature('HISTORY_SNIP')) {
  // ...
  snipTokensFreed = snipResult.tokensFreed
}
```

**为什么重要**：snip 删除了消息，但 assistant 消息中的 usage 字段仍然反映 snip 之前的上下文 token 数。如果 autoCompact 直接用 `tokenCountWithEstimation`，会看到过大的数字。传递 `snipTokensFreed` 让 autoCompact 的阈值检查使用 snip 之后的实际 token 数。

> **注意**：snipCompact 模块在发布版本中被消除（`feature('HISTORY_SNIP')` 返回 `false`），仅在内部版本可用。

### 模块组成（基于代码线索的反向推导）

snipCompact 的源代码已被 dead-code elimination 移除，但代码库中留下了大量调用痕迹、类型签名、注释和命名。以下基于这些线索反向推导出的实现细节。

#### 涉及的三个模块

| 模块 | 路径（已消除） | 导出符号 | 作用 |
|------|--------------|---------|------|
| snipCompact | `src/services/compact/snipCompact.js` | `snipCompactIfNeeded`, `isSnipRuntimeEnabled`, `isSnipMarkerMessage`, `shouldNudgeForSnips`, `SNIP_NUDGE_TEXT` | 核心 snip 逻辑 |
| snipProjection | `src/services/compact/snipProjection.js` | `projectSnippedView`, `isSnipBoundaryMessage` | API 视图过滤 |
| SnipTool | `src/tools/SnipTool/SnipTool.js` | `SnipTool`（工具实例） | Claude 调用的工具 |

#### 关键发现：boundary 消息存储 removedUuids

`src/utils/sessionStorage.ts:1982-1992` 揭示了核心数据模型：

```typescript
// boundary 消息携带 snipMetadata，记录被删除消息的 UUID 列表
type WithSnipMeta = { snipMetadata?: { removedUuids?: UUID[] } }

function applySnipRemovals(messages: Map<UUID, TranscriptMessage>): void {
  const toDelete = new Set<UUID>()
  for (const entry of messages.values()) {
    const removedUuids = (entry as WithSnipMeta).snipMetadata?.removedUuids
    if (!removedUuids) continue
    for (const uuid of removedUuids) toDelete.add(uuid)
  }
  // 删除对应消息，并重新链接 parentUuid 链
}
```

这意味着 snip 不是在每个消息上标记 `isSnipped`，而是**在 boundary 消息中记录删除了哪些 UUID**。会话恢复时通过 replay 这些 removals 重建状态。

#### snipCompactIfNeeded 推测实现

```typescript
// 基于 query.ts:403 调用签名和 QueryEngine.ts:1281 force 选项反向推导
export function snipCompactIfNeeded(
  messages: Message[],
  options?: { force?: boolean }
): { messages: Message[]; tokensFreed: number; boundaryMessage?: Message } {
  if (!isSnipRuntimeEnabled()) {
    return { messages, tokensFreed: 0 }
  }

  // 1. 判断是否需要执行
  //    force=true 来自 SDK replay（QueryEngine.ts:1281），跳过阈值检查
  //    非 force 时检查 token 使用量是否超过某个百分比阈值
  const needsSnip = options?.force || shouldAutoSnip(messages)
  if (!needsSnip) return { messages, tokensFreed: 0 }

  // 2. 收集可删除消息的 UUID
  //    来源：Claude 通过 SnipTool 显式标记的 + 自动策略选择的
  const removableUuids = collectRemovableUuids(messages)
  if (removableUuids.size === 0) return { messages, tokensFreed: 0 }

  // 3. 计算释放的 token 数（传递给 autoCompact 阈值检查）
  const tokensFreed = calculateTokensFreed(messages, removableUuids)

  // 4. 创建 snip marker（内部跟踪消息，UI 中隐藏）
  //    Message.tsx:277-279: isSnipMarkerMessage → return null
  const marker = createSnipMarkerMessage(removableUuids)

  // 5. 创建 snip boundary（用户可见 + 携带 snipMetadata）
  const boundaryMessage = createSnipBoundaryMessage({
    removedUuids: [...removableUuids],
    tokensRemoved: tokensFreed,
  })

  // 6. 过滤消息，插入 marker 和 boundary
  const filtered = messages.filter(m => !removableUuids.has(m.uuid))
  return {
    messages: [...filtered, marker, boundaryMessage],
    tokensFreed,
    boundaryMessage,
  }
}
```

#### SnipTool 推测实现

SnipTool 是 Claude 可以调用的工具。`src/utils/messages.ts:196-205` 和 `1616-1638` 揭示了其交互机制——每个用户消息被注入 `[id:xxxxxx]` 标签，Claude 通过这些标签引用消息。

**短 ID 生成**（`src/utils/messages.ts:200-204`）：

```typescript
function deriveShortMessageId(uuid: string): string {
  const hex = uuid.replace(/-/g, '').slice(0, 10)
  return parseInt(hex, 16).toString(36).slice(0, 6)
}
// 例如 uuid "a1b2c3d4-e5f6-..." → 短 ID "m8k3f2"
```

Claude 看到的消息末尾带有标签：
```
Here is the file content of foo.ts...
[id:m8k3f2]
```

**工具定义**（基于 `buildTool` 模式和 `collapseReadSearch.ts:177-192` 的分类信息反向推导）：

```typescript
// src/tools/SnipTool/SnipTool.js
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'
import { SNIP_TOOL_NAME } from './prompt.js'

const inputSchema = z.strictObject({
  messageIds: z
    .array(z.string())
    .describe('The [id:xxx] values of messages to remove from context'),
  reason: z
    .string()
    .optional()
    .describe('Why these messages are no longer needed'),
})

export const SnipTool = buildTool({
  name: SNIP_TOOL_NAME,

  // collapseReadSearch.ts:177-192 分类为 "meta-operation absorbed silently"
  // 不打断 collapse group，verbose 模式才可见
  isCollapsible: true,
  isAbsorbedSilently: true,

  // 只读 — 不直接修改任何状态，只是记录意图
  isReadOnly() { return true },
  // 允许并发 — 多个 snip 调用可以并行
  isConcurrencySafe() { return true },

  get inputSchema() { return inputSchema },

  // prompt.js 导出工具描述和系统提示
  async description() { return DESCRIPTION },
  async prompt() { return PROMPT },

  async call({ messageIds, reason }, context) {
    // 1. 收集当前消息中所有可用短 ID → UUID 映射
    const availableIds = new Map<string, string>()
    for (const msg of context.messages) {
      if (msg.type === 'user' && !msg.isMeta) {
        const shortId = deriveShortMessageId(msg.uuid)
        availableIds.set(shortId, msg.uuid)
      }
    }

    // 2. 解析并验证 Claude 给出的短 ID
    const resolved: string[] = []
    const notFound: string[] = []
    for (const shortId of messageIds) {
      const uuid = availableIds.get(shortId)
      if (uuid) {
        resolved.push(uuid)
      } else {
        notFound.push(shortId)
      }
    }

    // 3. 返回结果（作为 tool_result，Claude 下一轮能看到）
    //    注意：这里不删除任何消息！
    return {
      content: [{
        type: 'text',
        text: JSON.stringify({ snipped: resolved.length, resolved, notFound, reason }),
      }],
    }
  },
})
```

**关键设计：SnipTool 不直接删除消息**。它只验证 ID 并返回确认，实际删除由下一轮 `snipCompactIfNeeded` 执行。原因有三：

1. **职责分离**：Tool 负责"意图"，snipCompact 负责"执行"。Tool 的 `call` 是异步的、可能被取消的，而 snipCompact 在 query loop 开始时同步执行，保证一致性。
2. **`force` 重放**：QueryEngine 的 snipReplay 需要在自己的 store 上重放 snip（`force: true`）。如果删除逻辑在 Tool 里，QueryEngine 就无法重放。
3. **token 计数时机**：`tokensFreed` 必须在 API 调用之前计算出来（传递给 autoCompact 的阈值检查）。如果在 Tool 的 `call` 里删除，时机不对——Tool 执行时已经在 API 循环中间了。

**snipCompactIfNeeded 如何消费 SnipTool 的结果**（基于 `microCompact.ts:226-241` 的 `collectCompactableToolIds` 模式推导）：

```typescript
// snipCompact.js 内部
function collectRemovableUuids(messages: Message[]): Set<string> {
  const removable = new Set<string>()

  for (const message of messages) {
    if (message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue

    for (const block of message.message.content) {
      // 找到 SnipTool 的 tool_use 调用（按工具名过滤）
      if (block.type === 'tool_use' && block.name === SNIP_TOOL_NAME) {
        // 从 tool input 中提取 Claude 选择的 messageIds
        const ids: string[] = block.input?.messageIds ?? []

        // 短 ID → UUID 映射
        for (const shortId of ids) {
          const uuid = resolveShortId(shortId, messages)
          if (uuid) removable.add(uuid)
        }
      }
    }
  }

  return removable
}
```

这和 `microCompact.ts` 的 `collectCompactableToolIds` 是完全一样的模式——遍历 assistant 消息中的 `tool_use` 块，按工具名过滤，提取参数。

**完整执行流程**：

```
轮次 N:
  Assistant 消息:
    tool_use {
      id: "toolu_abc123",
      name: "SnipTool",
      input: { messageIds: ["m8k3f2", "p9q1r2"], reason: "已分析完毕" }
    }

  User 消息（tool_result）:
    tool_result {
      tool_use_id: "toolu_abc123",
      content: '{"snipped":2,"resolved":["uuid-a","uuid-c"],"notFound":[]}'
    }

轮次 N+1 开始:
  snipCompactIfNeeded(messages):
    1. collectRemovableUuids():
       - 遍历 assistant 消息
       - 找到 SnipTool 的 tool_use (name === SNIP_TOOL_NAME)
       - 提取 input.messageIds = ["m8k3f2", "p9q1r2"]
       - 解析为 uuid-a, uuid-c
       - 返回 Set { "uuid-a", "uuid-c" }

    2. 过滤 messages:
       - 移除 uuid-a 和 uuid-c 对应的消息

    3. 创建 boundary 消息:
       {
         type: 'system',
         subtype: 'snip_boundary',  // 实际值在 excluded-strings.txt 中
         snipMetadata: { removedUuids: ["uuid-a", "uuid-c"] },
         tokensRemoved: 15000
       }

    4. yield boundaryMessage → 用户看到 "✂ 已压缩 15K tokens"
```

#### 提示词注入机制

SnipTool 的提示词分两层注入，时机不同，共同作用让 Claude 知道工具的存在、用法，以及何时应该使用。

**第一层：工具描述（每轮 API 调用都发送）**

`src/utils/api.ts:169-177`，`toolToAPISchema` 把 SnipTool 的 prompt 变成 API 工具定义的一部分：

```typescript
// src/utils/api.ts:169-177
base = {
  name: tool.name,
  description: await tool.prompt({
    getToolPermissionContext: options.getToolPermissionContext,
    tools: options.tools,
    agents: options.agents,
    allowedAgentTypes: options.allowedAgentTypes,
  }),
  input_schema,
}
```

`SnipTool.prompt()` 返回 `prompt.js` 中的 `PROMPT`，这个文本成为工具定义的 `description` 字段。Claude **每轮 API 调用**都能看到它——它是 `tools` 数组的一部分。这让 Claude 知道：
- 有 SnipTool 这个工具
- 可以传入 `[id:xxx]` 标签来引用消息
- 应该只删除已消费的消息

**第二层：Nudge 提示词（上下文变大时条件注入）**

这是 `src/utils/attachments.ts:3963-3982` 实现的条件注入，只在上下文增长到一定程度时触发。

触发链路：

```
query.ts:1580  每轮循环中调用 getAttachmentMessages()
       ↓
attachments.ts:934-939  收集 attachments 时，检查 context_efficiency
       ↓
attachments.ts:3963-3982  getContextEfficiencyAttachment()
  → isSnipRuntimeEnabled()？
  → shouldNudgeForSnips(messages)？
  → 返回 [{ type: 'context_efficiency' }]
       ↓
messages.ts:4148-4161  normalizeAttachmentForAPI() 处理 context_efficiency
  → 读取 snipCompact.js 导出的 SNIP_NUDGE_TEXT
  → wrapInSystemReminder(SNIP_NUDGE_TEXT)
  → 变成 <system-reminder> 包裹的 isMeta 用户消息
```

最终 Claude 看到的消息：

```xml
<system-reminder>
Your context is getting large. Consider using the SnipTool to remove
older messages you no longer need, such as completed file reads or
finished bash commands. Reference messages by their [id:xxx] tags.
</system-reminder>
```

这条消息是 `isMeta: true` 的用户消息——**UI 中不显示，只在 API 上下文中存在**。

**两层的分工**：

| 层 | 注入方式 | 时机 | Claude 看到什么 |
|---|---------|------|---------------|
| **工具描述** | `tools` 数组的 `description` 字段 | 每轮 API 调用 | "有一个 SnipTool，可以传 `[id:xxx]` 删除消息" |
| **Nudge** | `<system-reminder>` 包裹的 `isMeta` 用户消息 | `shouldNudgeForSnips()` 返回 true 时 | "你的上下文变大了，考虑用 SnipTool 清理" |

工具描述让 Claude **知道工具的存在和用法**，Nudge 让 Claude **知道现在应该用它了**。两者缺一不可。

**`shouldNudgeForSnips` 的节奏控制**（基于注释反向推导）：

`src/utils/attachments.ts:3957-3961` 注释说明了 nudge 的节奏：

> Context-efficiency nudge. Injected after every N tokens of growth without a snip. Pacing is handled entirely by shouldNudgeForSnips — the 10k interval resets on prior nudges, snip markers, snip boundaries, and compact boundaries.

即：每增长约 10K tokens 且期间没有执行过 snip 时注入一次 nudge。计时器在以下事件后重置：
- 之前的 nudge
- snip marker（内部 snip 事件）
- snip boundary（用户可见的压缩通知）
- compact boundary（autoCompact 的压缩通知）

这避免了每轮都提示用户，同时确保上下文持续增长时 Claude 会被周期性地提醒。

#### snipProjection 推测实现

```typescript
// API 发送前过滤被 snip 的消息
// src/utils/messages.ts:4648-4654 中调用
export function projectSnippedView(messages: Message[]): Message[] {
  // 1. 收集所有 boundary 消息记录的 removedUuids
  const removedUuids = new Set<string>()
  for (const msg of messages) {
    if (isSnipBoundaryMessage(msg)) {
      const uuids = msg.snipMetadata?.removedUuids ?? []
      for (const uuid of uuids) removedUuids.add(uuid)
    }
  }
  // 2. 过滤：删除被 snip 的消息 + 内部 marker
  return messages.filter(m =>
    !removedUuids.has(m.uuid) && !isSnipMarkerMessage(m)
  )
}
```

REPL UI 保留完整历史（`includeSnipped: true`），只在 API 路径中过滤。

#### Nudge 机制

```typescript
// src/utils/attachments.ts:3965-3983
// 当上下文效率低时，生成 { type: 'context_efficiency' } attachment
// 注入 SNIP_NUDGE_TEXT 提示 Claude 使用 SnipTool
export function shouldNudgeForSnips(messages: Message[]): boolean {
  // 考虑因素（从注释 "resets on prior nudges, snip markers, snip boundaries" 推断）：
  // - 距上次 nudge 的间隔（避免每轮都提示）
  // - tool result 数量或 token 使用百分比
  // - 近期是否已执行过 snip
}
```

#### 两种消息类型

| 类型 | 判断函数 | UI 行为 | 用途 |
|------|---------|---------|------|
| **snip marker** | `isSnipMarkerMessage`（snipCompact.js） | 隐藏（`return null`） | 内部跟踪，记录 snip 事件 |
| **snip boundary** | `isSnipBoundaryMessage`（snipProjection.js） | 渲染 `<SnipBoundaryMessage>` | 用户可见的压缩通知 + 携带 `snipMetadata.removedUuids` |

#### 完整数据流

```
第 N 轮：
  API 返回 → Claude 调用 SnipTool({ messageIds: ["m8k3f2"] })
           → SnipTool 标记 uuid-a 为待 snip

第 N+1 轮开始：
  snipCompactIfNeeded() → 发现有待 snip 消息
    → removableUuids = {uuid-a}
    → tokensFreed = 15000
    → 创建 boundary { snipMetadata: { removedUuids: ["uuid-a"] } }
    → yield boundaryMessage → 用户看到 "✂ 已压缩 15K tokens"
    → messagesForQuery = 过滤后的消息

  getMessagesAfterCompactBoundary()
    → projectSnippedView() → API 只看到过滤后的消息

会话保存：
  boundary 的 snipMetadata.removedUuids 写入磁盘

会话恢复：
  applySnipRemovals() 读取 removedUuids → 从消息 map 中删除
```

---

## Layer 2: microcompact（微压缩）

**源码**：`src/services/compact/microCompact.ts`

### 功能

在单轮内进行轻量压缩，主要处理工具结果。有两种触发路径：

### 2.1 时间触发微压缩（time-based MC）

当距离上一次 assistant 消息的时间超过阈值（默认 30 分钟）时，服务器端缓存的 prefix 会过期被重写，此时提前清理旧的工具结果可以减少重写量。

**清理逻辑**：
- 保留最近的 N 个工具结果（默认 N=1）
- 其余工具结果的内容被替换为 `[Old tool result content cleared]`
- 直接修改消息内容（不是 cache_edits 方式）

**触发检查**：

```typescript
// src/services/compact/microCompact.ts:422-444
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  // Require an explicit main-thread querySource
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) {
    return null
  }
  const gapMinutes =
    (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  return { gapMinutes, config }
}
```

**执行清理**：

```typescript
// src/services/compact/microCompact.ts:446-530
function maybeTimeBasedMicrocompact(
  messages: Message[],
  querySource: QuerySource | undefined,
): MicrocompactResult | null {
  const trigger = evaluateTimeBasedTrigger(messages, querySource)
  if (!trigger) {
    return null
  }
  const { gapMinutes, config } = trigger

  const compactableIds = collectCompactableToolIds(messages)

  // Floor at 1: slice(-0) returns the full array, and clearing ALL
  // results leaves the model with zero working context.
  const keepRecent = Math.max(1, config.keepRecent)
  const keepSet = new Set(compactableIds.slice(-keepRecent))
  const clearSet = new Set(compactableIds.filter(id => !keepSet.has(id)))

  if (clearSet.size === 0) {
    return null
  }

  let tokensSaved = 0
  const result: Message[] = messages.map(message => {
    // 直接修改消息内容，替换为占位符
    if (block.type === 'tool_result' && clearSet.has(block.tool_use_id)) {
      return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
    }
    return block
  })

  // 重置 cached-MC state，因为改变了 prompt 内容
  resetMicrocompactState()
  return { messages: result }
}
```

### 2.2 缓存微压缩（cached MC）

使用 API 的 `cache_edits` 机制删除工具结果，**不修改本地消息内容**，而是在 API 层添加 `cache_reference` 和 `cache_edits` 字段，从而保留服务器端缓存的 prefix。

**触发阈值**：
- `triggerThreshold`：工具结果数量超过此值时触发（默认 50）
- `keepRecent`：始终保留最近 N 个工具结果（默认 10）

**关键流程**：

```typescript
// src/services/compact/microCompact.ts:305-399
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  // 1. 收集可压缩的工具 ID（Read, Bash, Grep, Glob 等）
  const compactableToolIds = new Set(collectCompactableToolIds(messages))

  // 2. 注册工具结果到 cachedMCState
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (
          block.type === 'tool_result' &&
          compactableToolIds.has(block.tool_use_id) &&
          !state.registeredTools.has(block.tool_use_id)
        ) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  // 3. 获取待删除的工具 ID
  const toolsToDelete = mod.getToolResultsToDelete(state)

  if (toolsToDelete.length > 0) {
    // 4. 创建 cache_edits block，延迟到 API 响应后再产出 boundary message
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits
    }

    // 5. 捕获 baseline cumulative cache_deleted_input_tokens
    const lastAsst = messages.findLast(m => m.type === 'assistant')
    const baseline = lastAsst?.message.usage?.cache_deleted_input_tokens ?? 0

    return {
      messages,  // 消息不变！cache_reference 和 cache_edits 在 API 层添加
      compactionInfo: {
        pendingCacheEdits: {
          trigger: 'auto',
          deletedToolIds: toolsToDelete,
          baselineCacheDeletedTokens: baseline,
        },
      },
    }
  }

  return { messages }
}
```

### 缓存编辑机制详解

缓存微压缩的核心是利用 API 的 `cache_edits` 机制：

1. **不修改本地消息内容**：与时间触发的微压缩不同，cached MC 不改变 `messages` 数组
2. **`cache_edits` 在 API 层添加**：压缩指示器被发送到 API，API 在其缓存的 prefix 上执行编辑
3. **延迟 boundary message**：boundary message 延迟到 API 响应后，使用实际的 `cache_deleted_input_tokens`
4. **增量计算**：`baselineCacheDeletedTokens` 用于计算本次操作的增量 token

### 可压缩的工具类型

```typescript
// src/services/compact/microCompact.ts:41-50
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

---

## Layer 3: contextCollapse（上下文折叠）

**源码**：`src/services/contextCollapse/`（feature-gated，发布版本不可见）

### 发布版本状态

**不可用** — `src/services/contextCollapse/` 整个目录在发布版本中被消除。`feature('CONTEXT_COLLAPSE')` 永远返回 `false`。

### 功能

折叠累积的上下文视图，是一种更细粒度的上下文管理机制。与 snipCompact 不同，contextCollapse 不删除消息，而是将多个消息压缩合并为更少的视图。

### 实现细节（基于代码交互的反向推导）

由于源码被消除，以下是基于 `src/query.ts`、`src/services/compact/autoCompact.ts`、`src/services/compact/postCompactCleanup.ts` 中的调用点和代码注释进行的反向推导。

#### 核心架构：写时记录、读时投影

```typescript
// src/query.ts:433-439
// Nothing is yielded — the collapsed view is a read-time projection
// over the REPL's full history. Summary messages live in the collapse
// store, not the REPL array. This is what makes collapses persist
// across turns: projectView() replays the commit log on every entry.
// Within a turn, the view flows forward via state.messages at the
// continue site (query.ts:1192), and the next projectView() no-ops
// because the archived messages are already gone from its input.
```

这说明 contextCollapse 使用了**写时记录、读时投影**（write-ahead log + read-time projection）的模式：

1. **Commit log**：每次工具调用、消息发送都追加到 commit log（模块级共享状态，跨 fork 共享）
2. **Collapse store**：累积到阈值时，生成折叠视图存入 collapse store
3. **projectView()**：每次进入时重放 commit log 生成当前视图
4. **跨轮次持久化**：因为摘要存在 collapse store 而不是消息数组，所以退出再进入仍然有效
5. **轮次内推进**：同一次交互中，view 通过 `state.messages` 向前推进

#### applyCollapsesIfNeeded

```typescript
// src/query.ts:440-447
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

关键观察：`applyCollapsesIfNeeded` 返回的 `collapseResult.messages` 是"折叠后的视图"，它**不 yield 任何消息**给用户（对比 autoCompact 会 yield boundary message）。

#### 阈值设计

从注释可知：
- **commit-start**：90% — 开始提交折叠
- **blocking**：95% — 触发阻塞

这意味着 contextCollapse 在上下文达到 90% 时开始做折叠，积累到 95% 才阻塞。这比 autoCompact 的触发阈值（~93% 减去 13,000 buffer）更早介入。

#### 与 autoCompact 的互斥关系

```typescript
// src/services/compact/autoCompact.ts:215-222
if (feature('CONTEXT_COLLAPSE')) {
  const { isContextCollapseEnabled } =
    require('../contextCollapse/index.js')
  if (isContextCollapseEnabled()) {
    return false  // 抑制 autoCompact
  }
}
```

当 contextCollapse 启用时，autoCompact 被完全抑制。原因：

> autocompact firing at effective-13k (~93% of effective) sits right between collapse's commit-start (90%) and blocking (95%), so it would race collapse and usually win, nuking granular context that collapse was about to save.

即 autoCompact 的触发阈值（93%）正好在 contextCollapse 的提交阈值（90%）和阻塞阈值（95%）之间，两者会竞争，autoCompact 会先触发并摧毁 contextCollapse 本来要保存的细粒度上下文。

#### 错误恢复：recoverFromOverflow

当 API 返回 413 时，contextCollapse 有专门的恢复路径：

```typescript
// src/query.ts:1086-1103
if (
  feature('CONTEXT_COLLAPSE') &&
  contextCollapse &&
  state.transition?.reason !== 'collapse_drain_retry'
) {
  const drained = contextCollapse.recoverFromOverflow(
    messagesForQuery,
    querySource,
  )
  if (drained.committed > 0) {
    // drain staged collapses 后重试
  }
}
```

说明 contextCollapse 有自己的**溢出恢复机制**，会在 413 时 drain staged collapses，然后重试。如果重试仍然 413，才 fall through 到 reactiveCompact。

#### resetContextCollapse 的危险性

```typescript
// src/services/compact/postCompactCleanup.ts:42-48
if (feature('CONTEXT_COLLAPSE')) {
  require('../contextCollapse/index.js').resetContextCollapse()
}
```

当 autoCompact 成功压缩后会调用 `resetContextCollapse()`。注释警告：

> autocompact fires, runPostCompactCleanup calls resetContextCollapse() which destroys the MAIN thread's committed log (module-level state shared across forks)

这说明 contextCollapse 的 commit log 是**模块级共享状态**，跨 fork 共享。autoCompact 调用 `resetContextCollapse()` 会摧毁主线程的 committed log——这正是为什么 `shouldAutoCompact` 中对 `marble_origami`（ctx-agent）做了特殊处理。

#### 与 reactiveCompact 的关系

```typescript
// src/query.ts:800-810
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (
    contextCollapse?.isWithheldPromptTooLong(
      message,
      isPromptTooLongMessage,
      querySource,
    )
  ) {
    withheld = true
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
```

两者都有 `isWithheldPromptTooLong` 判断，且是**独立工作**的。注释说："Either subsystem's withhold is sufficient — they're independent so turning one off doesn't break the other's recovery path."

#### 设计意图总结

| 方面 | 设计 |
|------|------|
| **架构** | Write-ahead log + read-time projection，折叠视图存在独立 store |
| **持久化** | 摘要存 collapse store 不在消息数组，跨会话保持 |
| **阈值** | 90% 提交，95% 阻塞 |
| **与 autoCompact** | 互斥，collapse 启用时 autoCompact 被抑制 |
| **错误恢复** | 有独立的 recoverFromOverflow，在 reactiveCompact 之前执行 |
| **状态管理** | 模块级共享状态，跨 fork 共享，autoCompact 会摧毁它 |

contextCollapse 本质上是一个**更智能的渐进式压缩系统**，它在上下文还远没到危险线时就开始做细粒度的折叠，保留更多有用信息，而不是等到最后被迫做一次性的总结。

---

## Layer 4: autoCompact（自动压缩总结）

**源码**：`src/services/compact/autoCompact.ts` + `src/services/compact/compact.ts`

### 功能

当上下文接近 token 限制时，对话历史被总结为一段摘要文本，用摘要消息替换原始历史。这是最后一层也是最彻底的一层压缩，**在发布版本中完全可用**。

### 自然语言工作流程

autoCompact 是这样工作的：

**1. 判断是否需要压缩**

每一轮对话结束后，系统会计算当前上下文的 token 使用量。如果超过了阈值——这个阈值 = 模型上下文窗口大小 - 20,000（预留输出空间）- 13,000（buffer）——就会触发自动压缩。例如，对于 200K 上下文的模型，阈值大约在 167,000 tokens 左右。

在判断时，系统会考虑 snipCompact 已经释放的 token 数量，确保不会重复计算。

**2. 优先尝试会话记忆压缩**

如果这是一次会话记忆压缩（session memory），系统会优先尝试一种更轻量的压缩方式。这种方式专门针对会话记忆设计，比通用压缩更高效。如果成功，就不需要执行完整的 autoCompact 流程。

**3. 电路断路器保护**

如果之前已经连续 3 次压缩失败（压缩后上下文仍然超限，或者 API 返回错误），系统会停止尝试，避免在已经"无药可救"的上下文上浪费 API 调用。这个设计是因为数据分析发现，有些会话竟然连续失败了 3000 多次调用，每天浪费约 25 万次 API 请求。

**4. 执行标准压缩（compactConversation）**

这是 autoCompact 的核心。当判断需要压缩时，系统执行以下步骤：

**第一步：准备压缩请求**。执行 PreCompact hooks（如果用户配置了的话），把用户可能提供的自定义指令合并进来。然后构建一个压缩提示词，这个提示词会告诉 LLM："请把之前的对话总结成一段摘要"。

**第二步：剥离无用内容**。在发送压缩请求之前，系统会剥离消息中的图像和文档内容。因为图像对生成摘要没有帮助，而且图像会导致压缩请求本身也超限（CC-1180 问题）。同时，技能发现/技能列表这类附件也会被剥离，因为压缩后它们会被重新注入。

**第三步：调用 LLM 生成摘要**。这是最关键的一步。系统把所有对话历史发送给 API，让另一个 LLM 调用生成一段摘要。这段摘要包含：对话的主要话题和目标、完成的关键步骤和决策、遗留的上下文和未完成事项、重要的文件路径和配置等。

压缩请求会尝试服用主对话已经缓存好的提示词 prefix，这样可以节省成本。如果缓存失效（cache miss），才会重新创建缓存。

如果压缩请求本身触发了 `prompt_too_long` 错误（压缩需要太多 token），系统会尝试一个兜底策略：按组删除最老的 API round（约 20%），然后重试，最多尝试 3 次。

**第四步：清理并重新注入**。压缩完成后，系统清空文件读取缓存（因为压缩后的上下文不包含之前的文件读取结果）。然后开始重建上下文：

- 重新注入最近访问的文件（最多 5 个，每个最多 5,000 tokens）
- 重新注入使用的技能内容（每个最多 5,000 tokens，总共最多 25,000 tokens）
- 重新注入计划文件内容（如果有）
- 重新注入工具、代理、MCP 的当前状态（让模型知道有哪些工具可用）
- 执行 SessionStart hooks（压缩本质上是一个新的"会话开始"）

**第五步：构建结果**。最终，系统输出：
- 一条边界消息，告知用户"之前的对话已经被压缩了，消耗了 X tokens"
- 一条摘要消息，内容是 LLM 生成的压缩摘要
- 所有重新注入的附件

**5. 压缩后上下文大小估算**

压缩后，系统会估算新的上下文大小，用于判断下一轮是否还需要压缩。如果估算结果仍然超过阈值，下一轮会再次触发压缩（连续压缩）。

---

## 压缩后的文件恢复

压缩会清空 `readFileState`（文件读取缓存），但系统会在压缩后重新注入最近访问的文件作为附件，减少模型重新读取文件的开销：

```typescript
// src/services/compact/compact.ts:1415-1464
export async function createPostCompactFileAttachments(
  readFileState: Record<string, { content: string; timestamp: number }>,
  toolUseContext: ToolUseContext,
  maxFiles: number = 5,
  preservedMessages: Message[] = [],
): Promise<AttachmentMessage[]> {
  // 1. 收集已在 preservedMessages 中可见的文件路径（避免重复注入）
  const preservedReadPaths = collectReadToolFilePaths(preservedMessages)

  // 2. 过滤并按最近访问时间排序
  const recentFiles = Object.entries(readFileState)
    .filter(
      file =>
        !shouldExcludeFromPostCompactRestore(file.filename, toolUseContext.agentId) &&
        !preservedReadPaths.has(expandPath(file.filename)),
    )
    .sort((a, b) => b.timestamp - a.timestamp)
    .slice(0, maxFiles)

  // 3. 并行生成文件附件
  const results = await Promise.all(
    recentFiles.map(async file => {
      const attachment = await generateFileAttachment(
        file.filename,
        { ...toolUseContext, fileReadingLimits: { maxTokens: POST_COMPACT_MAX_TOKENS_PER_FILE } },
        'tengu_post_compact_file_restore_success',
        'tengu_post_compact_file_restore_error',
        'compact',
      )
      return attachment ? createAttachmentMessage(attachment) : null
    }),
  )

  // 4. 按 token 预算过滤
  let usedTokens = 0
  return results.filter(result => {
    if (result === null) return false
    const attachmentTokens = roughTokenCountEstimation(jsonStringify(result))
    if (usedTokens + attachmentTokens <= POST_COMPACT_TOKEN_BUDGET) {  // 50,000
      usedTokens += attachmentTokens
      return true
    }
    return false
  })
}
```

### 文件恢复限制

```typescript
// src/services/compact/compact.ts:122-130
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5      // 最多恢复 5 个文件
export const POST_COMPACT_TOKEN_BUDGET = 50_000          // 总 token 预算
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000    // 每个文件最多 5,000 token
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000   // 每个技能最多 5,000 token
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000   // 技能总 token 预算
```

---

## 压缩边界消息

每次压缩后都会产出边界消息，通知用户哪些内容被压缩：

```typescript
// src/utils/messages.ts
export function createCompactBoundaryMessage(
  type: 'snip' | 'microcompact' | 'auto' | 'manual',
  tokensRemoved: number,
  lastPreCompactUuid?: string,
  userFeedback?: string,
  messagesSummarizedCount?: number,
): SystemCompactBoundaryMessage {
  return {
    type: 'system',
    subtype: 'compact_boundary',
    boundaryType: type,
    tokensRemoved,
    compactMetadata: {
      lastPreCompactMessageId: lastPreCompactUuid,
      userContext: userFeedback,
      messagesSummarizedCount,
    },
  }
}
```

---

## 关键设计决策

### 1. 电路断路器

```typescript
// 连续失败 3 次后停止尝试压缩
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### 2. 分层顺序

```
snipCompact (历史裁剪)
    ↓
microcompact (微压缩)
    ↓
contextCollapse (上下文折叠)
    ↓
autoCompact (自动压缩总结)
```

每层处理不同粒度的问题：snip 删除整条消息，microcompact 处理工具结果，contextCollapse 管理视图，autoCompact 做最终总结。

### 3. snipTokensFreed 传递

snip 释放的 token 反映到 autoCompact 阈值检查，否则 autoCompact 会用旧的（snip 之前的）token 计数判断是否需要压缩。

### 4. 图像剥离

```typescript
// 图像对摘要生成没有帮助，且可能导致压缩 API 调用本身超时
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    if (message.type !== 'user') return message
    // 将 image block 替换为 [image] 文本标记
    // 将 document block 替换为 [document] 文本标记
    // 同时处理嵌套在 tool_result 中的 image/document
  })
}
```

### 5. 缓存提示词共享（tengu_compact_cache_prefix）

压缩尝试复用主对话的缓存 prefix，命中率达到预期时避免重新创建缓存。测试确认禁用此特性会导致 98% 的缓存未命中。

### 6. 延迟 boundary message

cached microcompact 的 boundary message 延迟到 API 响应后，使用实际的 `cache_deleted_input_tokens` 而非客户端估算值。

### 7. PTL 重试（CC-1180）

当压缩请求本身触发 `prompt_too_long` 时，逐组删除最老的 API round 并重试，最多 3 次：

```typescript
// src/services/compact/compact.ts:243-291
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // 精确模式：累积直到覆盖 token gap
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 回退模式：删除 20% 的 groups
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  // 保持至少一个 group
  dropCount = Math.min(dropCount, groups.length - 1)
  const sliced = groups.slice(dropCount).flat()

  // 如果开头是 assistant 消息，插入一个 user marker（API 要求首条消息必须是 user）
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

### 8. Session Memory 压缩优先

在 autoCompact 中，session memory 压缩优先于标准压缩执行：

```typescript
// src/services/compact/autoCompact.ts:287-310
const sessionMemoryResult = await trySessionMemoryCompaction(
  messages,
  toolUseContext.agentId,
  recompactionInfo.autoCompactThreshold,
)
if (sessionMemoryResult) {
  setLastSummarizedMessageId(undefined)
  runPostCompactCleanup(querySource)
  notifyCompaction(querySource ?? 'compact', toolUseContext.agentId)
  markPostCompaction()
  return {
    wasCompacted: true,
    compactionResult: sessionMemoryResult,
  }
}
```

---

## 压缩结果构建

```typescript
// src/services/compact/compact.ts:330-338
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,           // 压缩边界标记
    ...result.summaryMessages,       // 摘要消息
    ...(result.messagesToKeep ?? []), // 保留的消息（部分压缩时）
    ...result.attachments,            // 附件（文件、技能、计划等）
    ...result.hookResults,           // hook 执行结果
  ]
}
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
│  │ ⚠️ 发布版本中被消除，仅内部版本可用                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 2: microcompact (CACHED_MICROCOMPACT)                  │   │
│  │                                                              │   │
│  │ 时间触发：距离上次 assistant 消息 > 30 分钟                   │   │
│  │   → 内容替换为 [Old tool result content cleared]             │   │
│  │                                                              │   │
│  │ 缓存触发：工具结果数量 > 阈值                                │   │
│  │   → cache_edits 删除工具结果，保留服务器缓存                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 3: contextCollapse (CONTEXT_COLLAPSE)                   │   │
│  │ 折叠累积的上下文视图                                          │   │
│  │ ⚠️ 发布版本中被消除，仅内部版本可用                           │   │
│  │ 当启用时，autoCompact 被抑制                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Layer 4: autoCompact                                          │   │
│  │                                                              │   │
│  │ 检查阈值 (effectiveContextWindow - 13,000)                   │   │
│  │ 如果超过阈值：                                               │   │
│  │                                                              │   │
│  │ 1. 优先尝试 sessionMemoryCompaction                          │   │
│  │ 2. 执行 compactConversation():                               │   │
│  │    a. 剥离图像/文档                                          │   │
│  │    b. 调用压缩模型生成摘要                                    │   │
│  │    c. 清空文件状态缓存                                        │   │
│  │    d. 生成文件/技能/计划附件                                  │   │
│  │    e. 重新注入工具 delta 附件                                 │   │
│  │    f. 执行 sessionStart hooks                                │   │
│  │    g. 构建压缩结果消息                                        │   │
│  │                                                              │   │
│  │ 电路断路器：连续失败 3 次后停止                               │   │
│  │ PTL 重试：压缩请求本身超时时，截断头部并重试（最多 3 次）      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  [boundary][summary][recent messages] → 继续循环                   │
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
| shouldAutoCompact | `src/services/compact/autoCompact.ts:160-239` |
| compactConversation | `src/services/compact/compact.ts:387-763` |
| streamCompactSummary | `src/services/compact/compact.ts:1136-1396` |
| stripImagesFromMessages | `src/services/compact/compact.ts:145-200` |
| buildPostCompactMessages | `src/services/compact/compact.ts:330-338` |
| truncateHeadForPTLRetry | `src/services/compact/compact.ts:243-291` |
| microcompactMessages | `src/services/compact/microCompact.ts:253-293` |
| cachedMicrocompactPath | `src/services/compact/microCompact.ts:305-399` |
| evaluateTimeBasedTrigger | `src/services/compact/microCompact.ts:422-444` |
| maybeTimeBasedMicrocompact | `src/services/compact/microCompact.ts:446-530` |
| createPostCompactFileAttachments | `src/services/compact/compact.ts:1415-1464` |
| compress 集成 | `src/query.ts:396-470` |
| snipCompact 集成 | `src/query.ts:401-409` |

---

*基于 Claude Code v2.1.88 源代码分析*
