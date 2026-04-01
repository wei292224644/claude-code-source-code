# Claude Code Harness Architecture - Technical Analysis

## Overview

Claude Code's "harness" is the core execution framework that orchestrates the interaction between the AI model and tools. This document provides a detailed technical analysis of how Claude Code v2.1.88 implements its agentic execution system.

---

## 1. Core Agent Loop

### 1.1 The `query()` Function

The heart of Claude Code's harness is the `query()` async generator function in `src/query.ts`:

```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

This is a **while-true loop** that continuously:
1. Sends messages to the Claude API
2. Receives streaming responses
3. Executes tools as they are needed
4. Continues until the task is complete

**Source**: `src/query.ts:219-239`

### 1.2 Loop Structure

```typescript
// eslint-disable-next-line no-constant-condition
while (true) {  // Infinite loop until exit condition
  // 1. Setup Phase (lines 337-648)
  //    - Snapshot config, check blocking limits
  //    - Setup streaming executor
  
  // 2. API Call Phase (lines 652-954)
  //    - Stream response from Claude API
  //    - Collect tool_use blocks as they stream
  
  // 3. Tool Execution Phase (lines 1363-1509)
  //    - Execute tools via StreamingToolExecutor or runTools()
  
  // 4. Recovery Phase (lines 1062-1176)
  //    - Handle prompt-too-long, max-output-tokens
  //    - Reactive compact
  
  // 5. Continuation Phase (lines 1714-1727)
  //    - Accumulate messages
  //    - Recurse with new state
}
```

**Source**: `src/query.ts:306-1728`

### 1.3 State Management

Cross-iteration mutable state:

```typescript
type State = {
  messages: Message[]                    // Conversation history
  toolUseContext: ToolUseContext       // Tool execution context
  autoCompactTracking | undefined      // Auto compression tracking
  maxOutputTokensRecoveryCount          // Output limit recovery
  hasAttemptedReactiveCompact          // Reactive compact attempt flag
  pendingToolUseSummary | undefined    // Tool summary pending
  stopHookActive | undefined           // Stop hooks status
  turnCount: number                    // Turn counter
  transition: Continue | undefined     // Continuation reason
}
```

**Source**: `src/query.ts:201-217`

---

## 2. Streaming Tool Execution

### 2.1 StreamingToolExecutor

The key innovation in Claude Code is **streaming tool execution** — tools begin executing as soon as their `tool_use` block appears in the stream, without waiting for the stream to complete.

```typescript
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 1. Find tool definition
    // 2. Check concurrency safety
    // 3. Add to execution queue
    // 4. Start processing immediately
    void this.processQueue()
  }
}
```

**Source**: `src/services/tools/StreamingToolExecutor.ts:40-124`

### 2.2 Concurrency Control Rules

```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

| Tool Type | Concurrency | Behavior |
|-----------|-------------|----------|
| Read-only (Glob, Grep, WebFetch) | ✅ Safe | Execute in parallel |
| Write operations (Bash, Edit) | ❌ Not safe | Execute serially |

**Source**: `src/services/tools/StreamingToolExecutor.ts:128-135`

### 2.3 Streaming Flow in query.ts

```typescript
for await (const message of deps.callModel({ ... })) {
  yield message  // Yield to UI immediately
  
  if (message.type === 'assistant') {
    // Extract and execute tools as they stream in
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)  // ← Immediate!
    }
    
    // Yield completed tool results immediately
    for (const result of streamingToolExecutor.getCompletedResults()) {
      yield result.message
    }
  }
}
```

**Source**: `src/query.ts:826-862`

### 2.4 Tool Status Tracking

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

**Source**: `src/services/tools/StreamingToolExecutor.ts:21-32`

---

## 3. The 12 Progressive Harness Mechanisms

Claude Code layers production features on the core agent loop:

| Layer | Name | Description | Source |
|-------|------|-------------|--------|
| **s01** | THE LOOP | Core while-true fetch-execute cycle | `src/query.ts` |
| **s02** | TOOL DISPATCH | Plugin-style tool registration | `src/Tool.ts`, `src/tools.ts` |
| **s03** | PLANNING | Plan-first execution via PlanMode | `EnterPlanModeTool`, `ExitPlanModeTool` |
| **s04** | SUB-AGENTS | Fork agents with fresh context | `AgentTool`, `forkSubagent.ts` |
| **s05** | KNOWLEDGE ON DEMAND | Lazy skill/memory injection | `SkillTool`, `memdir/` |
| **s06** | CONTEXT COMPRESSION | Three-layer compaction | `services/compact/` |
| **s07** | PERSISTENT TASKS | File-based task graph | `TaskCreateTool`, `TaskUpdateTool` |
| **s08** | BACKGROUND TASKS | Async daemon operations | `DreamTask`, `LocalShellTask` |
| **s09** | AGENT TEAMS | Multi-agent coordination | `TeamCreateTool`, `InProcessTeammateTask` |
| **s10** | TEAM PROTOCOLS | Inter-agent messaging | `SendMessageTool` |
| **s11** | AUTONOMOUS AGENTS | Idle-cycle auto-claim | `coordinatorMode.ts` |
| **s12** | WORKTREE ISOLATION | Per-agent directory isolation | `EnterWorktreeTool`, `ExitWorktreeTool` |

---

## 4. Tool System Architecture

### 4.1 Tool Interface

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
  // ... many optional methods
}
```

**Source**: `src/Tool.ts:362-400`

### 4.2 ToolUseContext

The context passed to every tool execution:

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
  // ... callbacks for progress, notifications, etc.
}
```

**Source**: `src/Tool.ts:158-300`

### 4.3 Built-in Tools Registry

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

**Source**: `src/tools.ts:193-251`

### 4.4 Tool Execution Flow

```typescript
// src/services/tools/toolExecution.ts

async function checkPermissionsAndCallTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
): Promise<ToolResult> {
  // 1. Validate input with Zod schema
  // 2. Validate with tool.validateInput()
  // 3. Run PreToolUse hooks
  // 4. Check permissions via canUseTool()
  // 5. Execute tool.call()
  // 6. Process result
  // 7. Run PostToolUse hooks
}
```

**Source**: `src/services/tools/toolExecution.ts:599-745`

---

## 5. Permission System

### 5.1 Multi-layer Permission Checks

```
User Input
    ↓
validateInput()          ← Input validation
    ↓
tool.validateInput()     ← Tool-specific validation
    ↓
PreToolUse Hooks        ← Can modify input, block execution
    ↓
canUseTool()            ← Permission decision
   ├── Rule-based        ← Deny/Allow/Ask rules
   ├── Classifier         ← Speculative Bash classifier
   └── Interactive       ← User prompt
    ↓
tool.call()             ← Actual execution
    ↓
PostToolUse Hooks       ← Post-execution hooks
```

**Source**: `src/utils/permissions/permissions.ts`

### 5.2 Permission Modes

```typescript
type PermissionMode = 
  | 'default'     // Normal mode with user prompts
  | 'auto'        // Automated decisions via classifier
  | 'plan'        // Plan mode restrictions
  | 'bypass'      // Skip permission checks (dangerous)
```

**Source**: `src/types/permissions.ts`

### 5.3 canUseTool Flow

```typescript
async function canUseTool(
  toolName: string,
  toolInput: unknown,
  context: ToolUseContext,
): Promise<PermissionResult> {
  // 1. Check rule-based permissions (deny/allow/ask)
  // 2. If 'allow' → return allowed
  // 3. If 'deny' → return denied
  // 4. If 'ask' → show interactive dialog OR
  //    - Coordinator: handleCoordinatorPermission()
  //    - Swarm worker: handleSwarmWorkerPermission()
  //    - Classifier: check speculative Bash classifier
  //    - Interactive: handleInteractivePermission()
}
```

**Source**: `src/hooks/useCanUseTool.tsx`

---

## 6. Hook System

### 6.1 Hook Event Types

```typescript
type HookEvent = 
  | 'PreToolUse'           // Before tool execution
  | 'PostToolUse'          // After successful tool
  | 'PostToolUseFailure'  // After tool failure
  | 'Stop'                 // End of turn
  | 'UserPromptSubmit'     // On user input
  | 'SessionStart'         // Session start
  | 'PermissionRequest'     // Permission request
  | 'PermissionDenied'     // Permission denied
  | 'SubagentStart'        // Subagent started
  | 'Elicitation'          // URL elicitation
```

**Source**: `src/types/hooks.ts`

### 6.2 Hook Execution Points

```typescript
// PreToolUse hooks - before tool execution
executePreToolHooks()  // Can modify input, block execution

// PostToolUse hooks - after successful tool
executePostToolHooks()  // Can modify output, block continuation

// Stop hooks - end of turn
executeStopHooks()      // Can prevent continuation
```

**Source**: `src/utils/hooks.ts`

---

## 7. Context Compression

### 7.1 Three-Layer Strategy

| Layer | Feature Flag | Description |
|-------|--------------|-------------|
| **snipCompact** | `HISTORY_SNIP` | Trims middle history messages |
| **microcompact** | `CACHED_MICROCOMPACT` | Light compression within a turn |
| **autoCompact** | (always on) | Full summarization on context overflow |

### 7.2 Compression Trigger

```typescript
// In query.ts, before API call:
if (needsCompact(autoCompactTracking)) {
  autoCompactTracking = autoCompactIfNeeded(messages, autoCompactTracking)
}

// Apply snip before microcompact
if (feature('HISTORY_SNIP')) {
  snipTokensFreed = applySnipCompact(messages)
}
```

**Source**: `src/services/compact/autoCompact.ts`

---

## 8. API Integration

### 8.1 The `callModel` Function

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

**Source**: `src/services/api/claude.ts:752-780`

### 8.2 API Request Building

```typescript
// src/services/api/claude.ts:paramsFromContext()

const params = {
  model: normalizeModelStringForAPI(options.model),
  messages: addCacheBreakpoints(messagesForAPI, ...),
  system,
  tools: allTools,
  tool_choice: options.toolChoice,
  max_tokens: maxOutputTokens,
  thinking,  // Extended thinking config
  betas: betasParams,  // Feature flags
}
```

**Source**: `src/services/api/claude.ts:1699-1720`

### 8.3 Actual API Call

```typescript
// src/services/api/claude.ts:1822-1831

const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal },
  )
  .withResponse()
```

Uses `@anthropic-ai/sdk` for API communication.

**Source**: `src/services/api/client.ts`

---

## 9. QueryEngine

### 9.1 QueryEngine Class

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

**Source**: `src/QueryEngine.ts:184-200`

### 9.2 submitMessage Flow

```
submitMessage(prompt)
    ↓
processUserInput()  ← Parse slash commands, attachments
    ↓
buildSystemPrompt() ← Include tools, skills, memory
    ↓
query() generator  ← Main agent loop
    ↓
yield messages as they arrive
    ↓
Track usage, permissions, errors
```

**Source**: `src/QueryEngine.ts:209-428`

---

## 10. State Management

### 10.1 AppStateStore

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  toolPermissionContext: ToolPermissionContext
  mcp: { clients, tools, commands, resources }
  plugins: { enabled, disabled, commands }
  tasks: { [taskId]: TaskState }
  sessionHooks: SessionHooksState
  // ... 100+ other fields
}>
```

**Source**: `src/state/AppStateStore.ts`

### 10.2 Store Pattern

React `createContext` + `useSyncExternalStore` for subscription:

```typescript
const store = createContext<StoreApi<AppState>>()
const AppStateProvider = ({ children }) => {
  const [state, setState] = useState(initialState)
  return <store.Provider value={state}>{children}</store.Provider>
}
```

**Source**: `src/state/store.ts`

---

## 11. Extended Thinking (思考机制)

### 11.1 Thinking Configuration

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }                    // Claude 4.6+ models
  | { type: 'enabled'; budgetTokens: number }  // Older models
  | { type: 'disabled' }
```

**Source**: `src/utils/thinking.ts:10-14`

### 11.2 Thinking in API Request

```typescript
// src/services/api/claude.ts:1596-1630

if (hasThinking && modelSupportsThinking(options.model)) {
  if (modelSupportsAdaptiveThinking(options.model)) {
    thinking = { type: 'adaptive' }  // New models
  } else {
    thinking = { budget_tokens: 16000, type: 'enabled' }  // Old models
  }
}
```

**Source**: `src/services/api/claude.ts:1596-1630`

### 11.3 Streaming Response Handling

```typescript
// src/services/api/claude.ts:2148-2160

switch (delta.type) {
  case 'thinking_delta':
    contentBlock.thinking += delta.thinking  // Accumulate thinking
    break
  case 'text_delta':
    contentBlock.text += delta.text
    break
  case 'tool_use_input_delta':
    contentBlock.input += delta.input
    break
}
```

**Source**: `src/services/api/claude.ts:2148-2160`

---

## 12. Source Code Index

### Core Loop

| File | Key Functions | Lines |
|------|--------------|-------|
| `src/query.ts` | `query()`, `queryLoop()` | 219-1728 |
| `src/QueryEngine.ts` | `submitMessage()`, class | 184-500 |

### Tool System

| File | Key Functions | Lines |
|------|--------------|-------|
| `src/Tool.ts` | `Tool` type, `buildTool()` | 362-695 |
| `src/tools.ts` | `getAllBaseTools()` | 193-251 |
| `src/services/tools/toolExecution.ts` | `runToolUse()`, permission checks | 599-1300 |
| `src/services/tools/StreamingToolExecutor.ts` | `addTool()`, `getCompletedResults()` | 40-200 |

### API Integration

| File | Key Functions | Lines |
|------|--------------|-------|
| `src/services/api/claude.ts` | `queryModelWithStreaming()`, `paramsFromContext()` | 752-1720 |
| `src/services/api/client.ts` | `getAnthropicClient()` | 100-300 |

### State & Hooks

| File | Key Functions | Lines |
|------|--------------|-------|
| `src/state/AppStateStore.ts` | `AppState` type | 1-200 |
| `src/types/hooks.ts` | `HookEvent` type | 1-100 |
| `src/utils/hooks.ts` | `executePreToolHooks()`, `executePostToolHooks()` | 100-400 |

### Context Compression

| File | Key Functions | Lines |
|------|--------------|-------|
| `src/services/compact/autoCompact.ts` | `autoCompactIfNeeded()` | 1-300 |
| `src/services/compact/snipCompact.ts` | `applySnipCompact()` | 1-200 |

---

## 13. Execution Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     User Input                                      │
│                   "Fix this bug"                                    │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   QueryEngine                                       │
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
│  - Snapshot config                                                 │
│  - Check blocking limits                                           │
│  - Prepare system prompt (memory, tools, skills)                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    API Streaming                                    │
│     for await (message of callModel()) {                          │
│       yield message  // To UI immediately                          │
│       if (tool_use block) {                                       │
│         streamingToolExecutor.addTool(block)  // Execute ASAP      │
│       }                                                            │
│     }                                                             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│              StreamingToolExecutor                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Tool Queue Processing                                         │  │
│  │                                                             │  │
│  │ Read(file) ──┬──→ Concurrent (parallel) ──→ Results         │  │
│  │ Glob(*)    ──┤                                             │  │
│  │ Grep(...)  ──┤                                             │  │
│  │ Bash(write) ──┴──→ Serial (wait for Read/Glob)             │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  Permission Checks                                  │
│                                                                     │
│  PreToolUse Hooks → canUseTool() → Interactive Dialog             │
│       ↓                    ↓                                       │
│  Modify input       Rule-based / Classifier                        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    tool.call()                                      │
│              Actual tool execution                                 │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                  PostToolUse Hooks                                 │
│              Modify output, block continuation                     │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                   Loop Continuation                                 │
│                                                                     │
│  messages += toolResults                                           │
│  turnCount++                                                       │
│                                                                     │
│  if (needsMoreTools) → continue                                    │
│  else → return { reason: 'completed' }                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 14. Key Design Patterns

### 14.1 AsyncGenerator Streaming
- API responses stream to UI immediately
- Tools execute as soon as `tool_use` block appears
- Results yield in order regardless of execution timing

### 14.2 Builder Factory (`buildTool()`)
- Provides safe defaults for tool definitions
- Validates input schemas
- Composes tool behaviors

### 14.3 Feature Flags + DCE
- `feature('FLAG')` from `bun:bundle` enables compile-time dead code elimination
- External builds strip 108 internal modules

### 14.4 Snapshot State
- `FileHistoryState` enables undo/redo for file operations
- Copy-on-write semantics for safe mutations

### 14.5 Discriminated Unions
- Type-safe message handling via `message.type`
- Exhaustive switch handling at compile time

---

## 15. Recovery Mechanisms

### 15.1 Prompt Too Long Recovery

```typescript
// src/query.ts:1062-1176

if (isWithheld413) {
  // Try context collapse drain first (cheap)
  if (contextCollapse?.recoverFromOverflow()) {
    state = { ...nextState, transition: { reason: 'collapse_drain_retry' } }
    continue
  }
  
  // Fall back to reactive compact (full summary)
  if (reactiveCompact?.tryReactiveCompact()) {
    state = { ...nextState, transition: { reason: 'reactive_compact_retry' } }
    continue
  }
  
  // No recovery available → surface error
  return { reason: 'prompt_too_long' }
}
```

### 15.2 Max Output Tokens Recovery

```typescript
if (isWithheldMaxOutputTokens(lastMessage)) {
  // Retry at higher max_tokens (64k instead of 8k)
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: `Output token limit hit. Resume directly...`
    })
    // Continue with recovery message
  }
}
```

**Source**: `src/query.ts:1185-1256`

---

## 16. Summary

Claude Code's harness is a sophisticated production-grade agent framework that:

1. **Streams everything** — API responses, tool executions, results
2. **Layers features** — 12 progressive mechanisms build on core loop
3. **Manages context** — Three-layer compression prevents overflow
4. **Enforces safety** — Multi-layer permission checks and hooks
5. **Recovers gracefully** — Multiple recovery paths for errors
6. **Coordinates agents** — Sub-agents, teams, inter-agent messaging

The core insight is that the `query()` async generator is the **single coordination point** for all these mechanisms, allowing them to be composed without modifying the core loop.

---

*Analysis based on Claude Code v2.1.88 source code*
