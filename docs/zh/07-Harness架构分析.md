# Claude Code Harness 架构 - 技术分析

## 概述

Claude Code 的 "harness" 是核心执行框架，用于协调 AI 模型与工具之间的交互。本文提供 Claude Code v2.1.88 代理执行系统的详细技术分析。

---

## 1. 核心代理循环

### 1.1 `query()` 函数

Claude Code harness 的核心是 `src/query.ts` 中的 `query()` 异步生成器函数：

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

这是一个 **while-true 循环**，持续执行：
1. 向 Claude API 发送消息
2. 接收流式响应
3. 在需要时立即执行工具
4. 持续运行直到任务完成

**源码**: `src/query.ts:219-239`

### 1.2 循环结构

```typescript
// eslint-disable-next-line no-constant-condition
while (true) {  // 无限循环，直到满足退出条件
  // 1. Setup Phase (lines 337-648)
  //    - 快照配置，检查阻塞限制
  //    - 设置流式执行器
  
  // 2. API Call Phase (lines 652-954)
  //    - 从 Claude API 流式响应
  //    - 流式收集 tool_use 块
  
  // 3. Tool Execution Phase (lines 1363-1509)
  //    - 通过 StreamingToolExecutor 或 runTools() 执行工具
  
  // 4. Recovery Phase (lines 1062-1176)
  //    - 处理 prompt-too-long、max-output-tokens
  //    - 响应式压缩
  
  // 5. Continuation Phase (lines 1714-1727)
  //    - 累积消息
  //    - 用新状态递归
}
```

**源码**: `src/query.ts:306-1728`

### 1.3 状态管理

跨迭代可变状态：

```typescript
type State = {
  messages: Message[]                    // 对话历史
  toolUseContext: ToolUseContext       // 工具执行上下文
  autoCompactTracking | undefined      // 自动压缩跟踪
  maxOutputTokensRecoveryCount          // 输出限制恢复计数
  hasAttemptedReactiveCompact          // 是否尝试过响应式压缩
  pendingToolUseSummary | undefined    // 待处理的工具摘要
  stopHookActive | undefined          // stop hooks 状态
  turnCount: number                   // 轮次计数器
  transition: Continue | undefined     // 继续原因
}
```

**源码**: `src/query.ts:201-217`

---

## 2. 流式工具执行

### 2.1 StreamingToolExecutor

Claude Code 的关键创新是**流式工具执行** — 工具在其 `tool_use` 块出现在流中时立即开始执行，无需等待流结束。

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 1. 查找工具定义
    // 2. 检查并发安全性
    // 3. 加入执行队列
    // 4. 立即开始处理
    void this.processQueue()
  }
}
```

**源码**: `src/services/tools/StreamingToolExecutor.ts:40-124`

### 2.2 并发控制规则

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

| 工具类型 | 并发安全 | 行为 |
|---------|---------|------|
| 只读（Glob, Grep, WebFetch） | ✅ 安全 | 并行执行 |
| 写操作（Bash, Edit） | ❌ 不安全 | 串行执行 |

**源码**: `src/services/tools/StreamingToolExecutor.ts:128-135`

### 2.3 query.ts 中的流式流程

```typescript
for await (const message of deps.callModel({ ... })) {
  yield message  // 立即 yield 给 UI
  
  if (message.type === 'assistant') {
    // 流式提取并执行工具
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)  // ← 立即执行！
    }
    
    // 立即 yield 已完成工具的结果
    for (const result of streamingToolExecutor.getCompletedResults()) {
      yield result.message
    }
  }
}
```

**源码**: `src/query.ts:826-862`

### 2.4 工具状态跟踪

```typescript
type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: 'queued' | 'executing' | 'completed' | 'yielded'
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
}
```

**源码**: `src/services/tools/StreamingToolExecutor.ts:21-32`

---

## 3. 12 层 Progressive Harness

Claude Code 在核心代理循环上分层生产特性：

| Layer | 名称 | 描述 | 源码 |
|-------|------|------|------|
| **s01** | THE LOOP | 核心 while-true fetch-execute 循环 | `src/query.ts` |
| **s02** | TOOL DISPATCH | 插件式工具注册 | `src/Tool.ts`, `src/tools.ts` |
| **s03** | PLANNING | 通过 PlanMode 的计划优先执行 | `EnterPlanModeTool`, `ExitPlanModeTool` |
| **s04** | SUB-AGENTS | 带新鲜上下文的分叉代理 | `AgentTool`, `forkSubagent.ts` |
| **s05** | KNOWLEDGE ON DEMAND | 懒加载技能/记忆注入 | `SkillTool`, `memdir/` |
| **s06** | CONTEXT COMPRESSION | 三层压缩 | `services/compact/` |
| **s07** | PERSISTENT TASKS | 基于文件的任务图 | `TaskCreateTool`, `TaskUpdateTool` |
| **s08** | BACKGROUND TASKS | 异步守护进程操作 | `DreamTask`, `LocalShellTask` |
| **s09** | AGENT TEAMS | 多代理协调 | `TeamCreateTool`, `InProcessTeammateTask` |
| **s10** | TEAM PROTOCOLS | 代理间消息传递 | `SendMessageTool` |
| **s11** | AUTONOMOUS AGENTS | 空闲周期自动认领 | `coordinatorMode.ts` |
| **s12** | WORKTREE ISOLATION | 每代理目录隔离 | `EnterWorktreeTool`, `ExitWorktreeTool` |

---

## 4. 工具系统架构

### 4.1 工具接口

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string
  aliases?: string[]
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  description(
    input: z.infer<Input>,
    options: { ... }
  ): Promise<string>
  readonly inputSchema: Input
  isConcurrencySafe?(input: Input): boolean
  isReadOnly?(input: Input): boolean
  checkPermissions?(input: Input, context: ToolUseContext): Promise<PermissionResult>
  // ... 许多可选方法
}
```

**源码**: `src/Tool.ts:362-400`

### 4.2 ToolUseContext

传递给每次工具执行的上下文：

```typescript
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    mcpClients: MCPServerConnection[]
    agentDefinitions: AgentDefinitionsResult
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  // ... 进度通知等回调
}
```

**源码**: `src/Tool.ts:158-300`

### 4.3 内置工具注册表

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool, GlobTool, GrepTool,
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool,
    // ... plus many conditional tools
  ]
}
```

**源码**: `src/tools.ts:193-251`

### 4.4 工具执行流程

```typescript
// src/services/tools/toolExecution.ts

async function checkPermissionsAndCallTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<ToolResult> {
  // 1. 用 Zod schema 验证输入
  // 2. 用 tool.validateInput() 验证
  // 3. 运行 PreToolUse hooks
  // 4. 通过 canUseTool() 检查权限
  // 5. 执行 tool.call()
  // 6. 处理结果
  // 7. 运行 PostToolUse hooks
}
```

**源码**: `src/services/tools/toolExecution.ts:599-745`

---

## 5. 权限系统

### 5.1 多层权限检查

```
用户输入
    ↓
validateInput()          ← 输入验证
    ↓
tool.validateInput()     ← 工具特定验证
    ↓
PreToolUse Hooks        ← 可修改输入，阻止执行
    ↓
canUseTool()            ← 权限决策
   ├── 基于规则          ← Deny/Allow/Ask 规则
   ├── 分类器           ← 推测性 Bash 分类器
   └── 交互式           ← 用户提示
    ↓
tool.call()             ← 实际执行
    ↓
PostToolUse Hooks       ← 执行后 hooks
```

**源码**: `src/utils/permissions/permissions.ts`

### 5.2 权限模式

```typescript
type PermissionMode = 
  | 'default'     // 带用户提示的正常模式
  | 'auto'        // 通过分类器自动决策
  | 'plan'        // Plan 模式限制
  | 'bypass'      // 跳过权限检查（危险）
```

**源码**: `src/types/permissions.ts`

### 5.3 canUseTool 流程

```typescript
async function canUseTool(
  toolName: string,
  toolInput: unknown,
  context: ToolUseContext,
): Promise<PermissionResult> {
  // 1. 检查基于规则的权限 (deny/allow/ask)
  // 2. 如果 'allow' → 返回允许
  // 3. 如果 'deny' → 返回拒绝
  // 4. 如果 'ask' → 显示交互式对话框 或
  //    - Coordinator: handleCoordinatorPermission()
  //    - Swarm worker: handleSwarmWorkerPermission()
  //    - 分类器: check speculative Bash classifier
  //    - 交互式: handleInteractivePermission()
}
```

**源码**: `src/hooks/useCanUseTool.tsx`

---

## 6. Hook 系统

### 6.1 Hook 事件类型

```typescript
type HookEvent = 
  | 'PreToolUse'           // 工具执行前
  | 'PostToolUse'          // 成功执行工具后
  | 'PostToolUseFailure'  // 工具失败后
  | 'Stop'                 // 回合结束时
  | 'UserPromptSubmit'     // 用户输入时
  | 'SessionStart'         // 会话开始时
  | 'PermissionRequest'     // 权限请求时
  | 'PermissionDenied'      // 权限被拒绝时
  | 'SubagentStart'         // 子代理启动时
  | 'Elicitation'           // URL 征求意见
```

**源码**: `src/types/hooks.ts`

### 6.2 Hook 执行点

```typescript
// PreToolUse hooks - 工具执行前
executePreToolHooks()  // 可修改输入，阻止执行

// PostToolUse hooks - 成功执行工具后
executePostToolHooks()  // 可修改输出，阻止继续

// Stop hooks - 回合结束时
executeStopHooks()      // 可阻止继续
```

**源码**: `src/utils/hooks.ts`

---

## 7. 上下文压缩

### 7.1 三层策略

| Layer | Feature Flag | 描述 |
|-------|--------------|------|
| **snipCompact** | `HISTORY_SNIP` | 裁剪中间历史消息 |
| **microcompact** | `CACHED_MICROCOMPACT` | 单轮内轻量压缩 |
| **autoCompact** | (始终启用) | 上下文溢出时完全总结 |

### 7.2 压缩触发

```typescript
// 在 query.ts，API 调用前：
if (needsCompact(autoCompactTracking)) {
  autoCompactTracking = autoCompactIfNeeded(messages, autoCompactTracking)
}

// 在 microcompact 前应用 snip
if (feature('HISTORY_SNIP')) {
  snipTokensFreed = applySnipCompact(messages)
}
```

**源码**: `src/services/compact/autoCompact.ts`

---

## 8. API 集成

### 8.1 `callModel` 函数

```typescript
// src/services/api/claude.ts:queryModelWithStreaming()

export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  thinkingConfig,
  tools,
  signal,
  options,
}): AsyncGenerator<StreamEvent | AssistantMessage, void> {
  return yield* withStreamingVCR(messages, function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}
```

**源码**: `src/services/api/claude.ts:752-780`

### 8.2 API 请求构建

```typescript
// src/services/api/claude.ts:paramsFromContext()

const params = {
  model: normalizeModelStringForAPI(options.model),
  messages: addCacheBreakpoints(messagesForAPI, ...),
  system,
  tools: allTools,
  tool_choice: options.toolChoice,
  max_tokens: maxOutputTokens,
  thinking,  // Extended thinking 配置
  betas: betasParams,  // Feature flags
}
```

**源码**: `src/services/api/claude.ts:1699-1720`

### 8.3 实际 API 调用

```typescript
// src/services/api/claude.ts:1822-1831

const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal },
  )
  .withResponse()
```

使用 `@anthropic-ai/sdk` 进行 API 通信。

**源码**: `src/services/api/client.ts`

---

## 9. QueryEngine

### 9.1 QueryEngine 类

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  
  async* submitMessage(prompt, options): AsyncGenerator<SDKMessage>
  interrupt(): void
  getMessages(): readonly Message[]
  getSessionId(): string
}
```

**源码**: `src/QueryEngine.ts:184-200`

### 9.2 submitMessage 流程

```
submitMessage(prompt)
    ↓
processUserInput()  ← 解析斜杠命令，附件
    ↓
buildSystemPrompt() ← 包含工具、技能、记忆
    ↓
query() generator  ← 主代理循环
    ↓
消息流式 yield
    ↓
跟踪使用量、权限、错误
```

**源码**: `src/QueryEngine.ts:209-428`

---

## 10. 状态管理

### 10.1 AppStateStore

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  toolPermissionContext: ToolPermissionContext
  mcp: { clients, tools, commands, resources }
  plugins: { enabled, disabled, commands }
  tasks: { [taskId]: TaskState }
  sessionHooks: SessionHooksState
  // ... 100+ 其他字段
}>
```

**源码**: `src/state/AppStateStore.ts`

### 10.2 Store 模式

React `createContext` + `useSyncExternalStore` 用于订阅：

```typescript
const store = createContext<StoreApi<AppState>>()
const AppStateProvider = ({ children }) => {
  const [state, setState] = useState(initialState)
  return <store.Provider value={state}>{children}</store.Provider>
}
```

**源码**: `src/state/store.ts`

---

## 11. Extended Thinking（思考机制）

### 11.1 Thinking 配置

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }                    // Claude 4.6+ 模型
  | { type: 'enabled'; budgetTokens: number }  // 旧模型
  | { type: 'disabled' }
```

**源码**: `src/utils/thinking.ts:10-14`

### 11.2 API 请求中的 Thinking

```typescript
// src/services/api/claude.ts:1596-1630

if (hasThinking && modelSupportsThinking(options.model)) {
  if (modelSupportsAdaptiveThinking(options.model)) {
    thinking = { type: 'adaptive' }  // 新模型
  } else {
    thinking = { budget_tokens: 16000, type: 'enabled' }  // 旧模型
  }
}
```

**源码**: `src/services/api/claude.ts:1596-1630`

### 11.3 流式响应处理

```typescript
// src/services/api/claude.ts:2148-2160

switch (delta.type) {
  case 'thinking_delta':
    contentBlock.thinking += delta.thinking  // 累积思考
    break
  case 'text_delta':
    contentBlock.text += delta.text
    break
  case 'tool_use_input_delta':
    contentBlock.input += delta.input
    break
}
```

**源码**: `src/services/api/claude.ts:2148-2160`

---

## 12. 源代码索引

### 核心循环

| 文件 | 关键函数 | 行号 |
|------|---------|------|
| `src/query.ts` | `query()`, `queryLoop()` | 219-1728 |
| `src/QueryEngine.ts` | `submitMessage()`, class | 184-500 |

### 工具系统

| 文件 | 关键函数 | 行号 |
|------|---------|------|
| `src/Tool.ts` | `Tool` type, `buildTool()` | 362-695 |
| `src/tools.ts` | `getAllBaseTools()` | 193-251 |
| `src/services/tools/toolExecution.ts` | `runToolUse()`, permission checks | 599-1300 |
| `src/services/tools/StreamingToolExecutor.ts` | `addTool()`, `getCompletedResults()` | 40-200 |

### API 集成

| 文件 | 关键函数 | 行号 |
|------|---------|------|
| `src/services/api/claude.ts` | `queryModelWithStreaming()`, `paramsFromContext()` | 752-1720 |
| `src/services/api/client.ts` | `getAnthropicClient()` | 100-300 |

### 状态和 Hooks

| 文件 | 关键函数 | 行号 |
|------|---------|------|
| `src/state/AppStateStore.ts` | `AppState` type | 1-200 |
| `src/types/hooks.ts` | `HookEvent` type | 1-100 |
| `src/utils/hooks.ts` | `executePreToolHooks()`, `executePostToolHooks()` | 100-400 |

### 上下文压缩

| 文件 | 关键函数 | 行号 |
|------|---------|------|
| `src/services/compact/autoCompact.ts` | `autoCompactIfNeeded()` | 1-300 |
| `src/services/compact/snipCompact.ts` | `applySnipCompact()` | 1-200 |

---

## 13. 执行流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     用户输入                                          │
│                   "修复这个 bug"                                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   QueryEngine                                        │
│              submitMessage() → processUserInput()                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                       query()                                       │
│         async generator, while(true) loop                          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    Setup Phase                                      │
│  - 快照配置                                                         │
│  - 检查阻塞限制                                                     │
│  - 准备系统提示词（记忆、工具、技能）                               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    API Streaming                                     │
│     for await (message of callModel()) {                          │
│       yield message  // 立即 yield 给 UI                           │
│       if (tool_use block) {                                       │
│         streamingToolExecutor.addTool(block)  // 尽快执行           │
│       }                                                            │
│     }                                                             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│              StreamingToolExecutor                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ 工具队列处理                                                  │  │
│  │                                                             │  │
│  │ Read(file) ──┬──→ 并发（并行）──→ 结果                      │  │
│  │ Glob(*)    ──┤                                             │  │
│  │ Grep(...)  ──┤                                             │  │
│  │ Bash(write) ──┴──→ 串行（等待 Read/Glob 完成）             │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  权限检查                                            │
│                                                                     │
│  PreToolUse Hooks → canUseTool() → 交互式对话框                   │
│       ↓                    ↓                                       │
│  修改输入       基于规则 / 分类器                                  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    tool.call()                                      │
│              实际工具执行                                          │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  PostToolUse Hooks                                 │
│              修改输出，阻止继续                                    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   循环继续                                          │
│                                                                     │
│  messages += toolResults                                          │
│  turnCount++                                                       │
│                                                                     │
│  if (needsMoreTools) → continue                                   │
│  else → return { reason: 'completed' }                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 14. 关键设计模式

### 14.1 AsyncGenerator 流式处理
- API 响应立即流式到 UI
- 工具在 `tool_use` 块出现时立即执行
- 结果按顺序 yield，无论执行时间

### 14.2 Builder Factory (`buildTool()`)
- 为工具定义提供安全默认值
- 验证输入 schema
- 组合工具行为

### 14.3 Feature Flags + DCE
- `feature('FLAG')` 来自 `bun:bundle`，启用编译时死代码消除
- 外部构建剥离 108 个内部模块

### 14.4 Snapshot State
- `FileHistoryState` 支持文件操作撤销/重做
- 写时复制语义保证安全变更

### 14.5 Discriminated Unions
- 通过 `message.type` 实现类型安全的消息处理
- 编译时穷尽 switch 处理

---

## 15. 恢复机制

### 15.1 Prompt Too Long 恢复

```typescript
// src/query.ts:1062-1176

if (isWithheld413) {
  // 先尝试上下文 collapse 排水（便宜）
  if (contextCollapse?.recoverFromOverflow()) {
    state = { ...nextState, transition: { reason: 'collapse_drain_retry' } }
    continue
  }
  
  // 回退到响应式压缩（完全总结）
  if (reactiveCompact?.tryReactiveCompact()) {
    state = { ...nextState, transition: { reason: 'reactive_compact_retry' } }
    continue
  }
  
  // 无恢复可用 → 暴露错误
  return { reason: 'prompt_too_long' }
}
```

### 15.2 Max Output Tokens 恢复

```typescript
if (isWithheldMaxOutputTokens(lastMessage)) {
  // 用更高 max_tokens 重试（64k 而非 8k）
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly...`
    })
    // 用恢复消息继续
  }
}
```

**源码**: `src/query.ts:1185-1256`

---

## 16. 总结

Claude Code 的 harness 是一个复杂的生产级代理框架：

1. **流式处理一切** — API 响应、工具执行、结果
2. **分层特性** — 12 种渐进机制构建在核心循环之上
3. **管理上下文** — 三层压缩防止溢出
4. **强制安全** — 多层权限检查和 hooks
5. **优雅恢复** — 多种错误恢复路径
6. **协调代理** — 子代理、团队、代理间消息

核心洞察是 `query()` 异步生成器是所有这些机制的**单一协调点**，允许它们在不改写核心循环的情况下组合。

---

*基于 Claude Code v2.1.88 源代码分析*
