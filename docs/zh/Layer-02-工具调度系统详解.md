# Layer 02: TOOL DISPATCH — 工具调度系统

## 概述

**TOOL DISPATCH** 是 Claude Code 的插件式工具注册与调度系统。它位于 `src/tools.ts` 和 `src/Tool.ts`，通过统一的 `Tool` 接口注册和调用约 50 个内置工具。

---

## 源码位置

| 项目 | 路径 |
|------|------|
| 工具注册表 | `src/tools.ts` |
| 工具接口定义 | `src/Tool.ts` |
| 工具执行引擎 | `src/services/tools/toolExecution.ts` |
| 流式工具执行器 | `src/services/tools/StreamingToolExecutor.ts` |
| 工具编排 | `src/services/tools/toolOrchestration.ts` |
| 工具权限钩子 | `src/services/tools/toolHooks.ts` |

---

## 工具注册表

### 全部基础工具清单

```typescript
// src/tools.ts:193-251
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 嵌入式搜索工具（ripgrep）可用时隐藏 GlobTool/GrepTool
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // ... 更多条件工具
  ]
}
```

### 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | FileRead, FileWrite, FileEdit, Glob, Grep | 文件 I/O 和搜索 |
| **执行** | Bash, PowerShell | Shell 命令执行 |
| **Web** | WebFetch, WebSearch | 网络访问 |
| **任务** | TaskCreate, TaskUpdate, TaskList, TaskGet | 任务管理 |
| **Agent** | Agent, Fork | 子代理创建 |
| **计划** | EnterPlanMode, ExitPlanMode, TodoWrite | 计划模式 |
| **工作区** | EnterWorktree, ExitWorktree | Git Worktree 隔离 |
| **团队** | TeamCreate, TeamDelete, SendMessage | 多代理协作 |
| **其他** | NotebookEdit, MCP, Skill, AskUserQuestion | 各种功能 |

### 条件注册工具

```typescript
// src/tools.ts:16-155
// 条件编译示例：ANT 内部工具
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null

// 特性门控工具
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const WebBrowserTool = feature('WEB_BROWSER_TOOL')
  ? require('./tools/WebBrowserTool/WebBrowserTool.js').WebBrowserTool
  : null
```

---

## Tool 接口定义

### 核心类型

```typescript
// src/Tool.ts:362-695
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 标识
  readonly name: string
  aliases?: string[]  // 向后兼容的别名

  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  description(
    input: z.infer<Input>,
    options: {...},
  ): Promise<string>

  readonly inputSchema: Input

  // 能力标志
  isConcurrencySafe(input: z.infer<Input>): boolean
  isEnabled(): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  interruptBehavior?(): 'cancel' | 'block'

  // 权限
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>

  // 渲染
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  // ... 更多渲染方法
}
```

### ToolUseContext - 工具执行上下文

```typescript
// src/Tool.ts:158-300
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools                    // 可用工具列表
    verbose: boolean
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  // ...
  messages: Message[]                // 对话历史
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  // ...
}
```

---

## 工具构建工厂

### buildTool 函数

```typescript
// src/Tool.ts:783-792
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}

// 默认值定义
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (...) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}
```

---

## 工具发现与过滤

### assembleToolPool - 完整工具池

```typescript
// src/tools.ts:345-367
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)

  // 过滤 MCP 工具中的拒绝规则
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 按名称排序以保证 prompt-cache 稳定性
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

### filterToolsByDenyRules - 权限过滤

```typescript
// src/tools.ts:262-269
export function filterToolsByDenyRules<T>(tools: readonly T[], permissionContext: ToolPermissionContext): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool))
}
```

---

## 工具执行流程

### 两种执行模式

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Layer 01: THE LOOP                              │
│                                                                     │
│  Phase 2: API CALL PHASE                                            │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ for await (const message of deps.callModel(...)) {           │ │
│  │   if (streamingToolExecutor) {                                │ │
│  │     streamingToolExecutor.addTool(toolBlock, assistantMessage) │ │
│  │   }                                                          │ │
│  │ }                                                            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              ↓                                     │
│  Phase 4: TOOL EXECUTION PHASE                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ const toolUpdates = streamingToolExecutor                     │ │
│  │   ? streamingToolExecutor.getRemainingResults()                │ │
│  │   : runTools(toolUseBlocks, ...)                              │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### StreamingToolExecutor - 流式工具执行

```typescript
// src/services/tools/StreamingToolExecutor.ts:40-62
export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private hasErrored = false
  private discarded = false

  constructor(
    private readonly toolDefinitions: Tools,
    private readonly canUseTool: CanUseToolFn,
    toolUseContext: ToolUseContext,
  ) {}

  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    const toolDefinition = findToolByName(this.toolDefinitions, block.name)
    // ... 验证工具存在
    // ... 检查 isConcurrencySafe
    this.tools.push({ id: block.id, block, status: 'queued', ... })
    void this.processQueue()
  }
}
```

### 并发控制

```typescript
// src/services/tools/StreamingToolExecutor.ts:129-151
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}

private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)
    } else {
      // 非并发安全工具需要独占访问
      if (!tool.isConcurrencySafe) break
    }
  }
}
```

### 工具执行

```typescript
// src/services/tools/StreamingToolExecutor.ts:265-396
private async executeTool(tool: TrackedTool): Promise<void> {
  tool.status = 'executing'

  const generator = runToolUse(
    tool.block,
    tool.assistantMessage,
    this.canUseTool,
    { ...this.toolUseContext, abortController: toolAbortController },
  )

  for await (const update of generator) {
    if (update.message.type === 'progress') {
      tool.pendingProgress.push(update.message)  // 进度消息立即产出
    } else {
      messages.push(update.message)  // 最终结果
    }
  }
}
```

---

## runToolUse - 工具执行核心

```typescript
// src/services/tools/toolExecution.ts:337-490
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  const toolName = toolUse.name

  // 1. 查找工具定义
  let tool = findToolByName(toolUseContext.options.tools, toolName)
  if (!tool) {
    // 尝试通过别名查找（兼容旧版名称）
    const fallbackTool = findToolByName(getAllBaseTools(), toolName)
    if (fallbackTool?.aliases?.includes(toolName)) {
      tool = fallbackTool
    }
  }

  // 2. 工具不存在
  if (!tool) {
    yield { message: createUserMessage({ content: [{ type: 'tool_result', content: `No such tool: ${toolName}`, is_error: true, ... }] }) }
    return
  }

  // 3. 检查中止信号
  if (toolUseContext.abortController.signal.aborted) {
    yield { message: createToolResultStopMessage(toolUse.id) }
    return
  }

  // 4. 执行权限检查和工具调用
  for await (const update of streamedCheckPermissionsAndCallTool(...)) {
    yield update
  }
}
```

---

## 权限检查流程

### checkPermissionsAndCallTool

```typescript
// src/services/tools/toolExecution.ts:599+
async function checkPermissionsAndCallTool(
  tool: Tool,
  toolUseID: string,
  input: { [key: string]: boolean | string | number },
  toolUseContext: ToolUseContext,
  canUseTool: CanUseToolFn,
  // ...
): Promise<MessageUpdateLazy[]> {
  // 1. validateInput - 工具自定义验证
  if (tool.validateInput) {
    const validation = await tool.validateInput(input, toolUseContext)
    if (!validation.result) {
      return [createErrorResult(validation.message)]
    }
  }

  // 2. canUseTool - 权限系统检查
  const canUse = await canUseTool(tool.name, input, toolUseContext)
  if (!canUse) {
    return [createErrorResult('Permission denied')]
  }

  // 3. 执行工具
  const result = await tool.call(input, toolUseContext, canUseTool, parentMessage, onProgress)

  // 4. 后置钩子
  await runPostToolUseHooks(...)

  return result
}
```

---

## 工具执行结果处理

### ToolResult 结构

```typescript
// src/Tool.ts:321-336
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

### 工具结果渲染

每个工具负责自己的渲染：

```typescript
// src/Tool.ts:566-580
renderToolResultMessage?(
  content: Output,
  progressMessagesForMessage: ProgressMessage<P>[],
  options: {
    style?: 'condensed'
    theme: ThemeName
    tools: Tools
    verbose: boolean
    isTranscriptMode?: boolean
    isBriefOnly?: boolean
    input?: unknown
  },
): React.ReactNode
```

---

## 工具生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                        工具生命周期                                   │
│                                                                     │
│  1. REGISTER (getAllBaseTools)                                      │
│     └─ Tool 定义注册到工具注册表                                     │
│                                                                     │
│  2. DISCOVER (assembleToolPool)                                     │
│     ├─ 过滤 deny rules                                              │
│     ├─ 过滤 REPL mode                                               │
│     ├─ 合并 MCP tools                                               │
│     └─ 返回可用工具列表                                              │
│                                                                     │
│  3. API REQUEST (callModel)                                         │
│     └─ 工具 schema 发送给模型                                        │
│                                                                     │
│  4. STREAM (StreamingToolExecutor)                                   │
│     ├─ 模型返回 tool_use 块                                          │
│     ├─ addTool() 添加工具到队列                                      │
│     └─ processQueue() 并发执行                                       │
│                                                                     │
│  5. EXECUTE (runToolUse)                                            │
│     ├─ validateInput()                                               │
│     ├─ canUseTool() 权限检查                                         │
│     ├─ tool.call() 执行                                              │
│     └─ runPostToolUseHooks() 后置钩子                                │
│                                                                     │
│  6. RENDER (renderToolResultMessage)                                │
│     └─ 渲染工具结果到 UI                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计模式

### 1. buildTool 工厂模式

所有工具通过 `buildTool()` 创建，保证一致的默认行为：
- `isEnabled: () => true` - 默认启用
- `isConcurrencySafe: () => false` - 默认非并发安全
- `isReadOnly: () => false` - 默认可写

### 2. 插件式工具注册

工具注册是静态的（编译时确定），但通过 `feature()` 条件编译实现条件加载。

### 3. 流式执行 vs 批量执行

```typescript
// 流式执行（默认）
const streamingToolExecutor = new StreamingToolExecutor(tools, canUseTool, context)
streamingToolExecutor.addTool(block, message)  // 边接收边添加
for await (const result of streamingToolExecutor.getRemainingResults()) {
  yield result.message
}

// 批量执行（fallback）
for await (const update of runTools(toolUseBlocks, ...)) {
  yield update.message
}
```

### 4. 并发安全控制

- **并发安全工具**：可与其他并发安全工具并行执行
- **非并发安全工具**：需要独占访问，顺序执行

### 5. 结果大小限制

```typescript
// src/Tool.ts:466
maxResultSizeChars: number  // 超过此限制，结果持久化到磁盘
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| getAllBaseTools | `src/tools.ts:193-251` |
| getTools | `src/tools.ts:271-327` |
| assembleToolPool | `src/tools.ts:345-367` |
| filterToolsByDenyRules | `src/tools.ts:262-269` |
| Tool 类型定义 | `src/Tool.ts:362-695` |
| ToolUseContext | `src/Tool.ts:158-300` |
| buildTool 工厂 | `src/Tool.ts:783-792` |
| TOOL_DEFAULTS | `src/Tool.ts:757-769` |
| findToolByName | `src/Tool.ts:358-360` |
| runToolUse | `src/services/tools/toolExecution.ts:337-490` |
| StreamingToolExecutor | `src/services/tools/StreamingToolExecutor.ts:40-500` |
| addTool | `src/services/tools/StreamingToolExecutor.ts:76-124` |
| processQueue | `src/services/tools/StreamingToolExecutor.ts:140-151` |
| executeTool | `src/services/tools/StreamingToolExecutor.ts:265-396` |
| checkPermissionsAndCallTool | `src/services/tools/toolExecution.ts:599+` |

---

## 与其他层的关系

```
Layer 01: THE LOOP
    │
    └── Layer 02: TOOL DISPATCH
            │
            ├── Layer 03: PLANNING
            │       └─ ExitPlanModeTool, TodoWriteTool 使用工具接口
            │
            ├── Layer 04: SUB-AGENTS
            │       └─ AgentTool 创建新的 query 调用
            │
            ├── Layer 05: KNOWLEDGE ON DEMAND
            │       └─ SkillTool 触发技能加载
            │
            └── Layer 07: PERSISTENT TASKS
                    └─ TaskCreateTool, TaskUpdateTool 等使用任务接口
```

---

*基于 Claude Code v2.1.88 源代码分析*
