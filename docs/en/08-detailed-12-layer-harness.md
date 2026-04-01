# Claude Code 12-Layer Progressive Harness - Detailed Analysis

## Overview

Claude Code's harness uses a **layered architecture** where each layer builds upon the previous one, incrementally adding capabilities without modifying the core loop. This design enables feature decoupling, independent evolution, while maintaining simplicity in core logic.

```
┌─────────────────────────────────────────────────────────────┐
│                      Layer 12: WORKTREE ISOLATION          │
│                      (Per-agent directory isolation)          │
├─────────────────────────────────────────────────────────────┤
│                      Layer 11: AUTONOMOUS AGENTS            │
│                      (Idle-cycle auto-claim)                 │
├─────────────────────────────────────────────────────────────┤
│                      Layer 10: TEAM PROTOCOLS               │
│                      (Inter-agent messaging)                │
├─────────────────────────────────────────────────────────────┤
│                      Layer 09: AGENT TEAMS                 │
│                      (Multi-agent coordination)             │
├─────────────────────────────────────────────────────────────┤
│                      Layer 08: BACKGROUND TASKS            │
│                      (Async daemon operations)              │
├─────────────────────────────────────────────────────────────┤
│                      Layer 07: PERSISTENT TASKS             │
│                      (File-based task graph)               │
├─────────────────────────────────────────────────────────────┤
│                      Layer 06: CONTEXT COMPRESSION          │
│                      (Three-layer compaction)               │
├─────────────────────────────────────────────────────────────┤
│                      Layer 05: KNOWLEDGE ON DEMAND         │
│                      (Lazy skill/memory injection)        │
├─────────────────────────────────────────────────────────────┤
│                      Layer 04: SUB-AGENTS                  │
│                      (Fork agents)                         │
├─────────────────────────────────────────────────────────────┤
│                      Layer 03: PLANNING                    │
│                      (Plan-first execution)                 │
├─────────────────────────────────────────────────────────────┤
│                      Layer 02: TOOL DISPATCH               │
│                      (Plugin-style tool registration)      │
├─────────────────────────────────────────────────────────────┤
│                      Layer 01: THE LOOP                    │
│                      (Core while-true loop)               │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 01: THE LOOP — Core Loop

### What It Is

The **heart** of the harness, located in `src/query.ts`. A `while(true)` async generator that continuously executes the fetch-execute cycle until the task completes or an exit condition is met.

### Core Code

```typescript
// src/query.ts:219-239
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
> {
  // ...
  while (true) {
    // 1. Setup Phase - Initialize config
    // 2. API Call Phase - Call Claude API
    // 3. Tool Execution Phase - Execute tools
    // 4. Recovery Phase - Error recovery
    // 5. Continuation Phase - Loop continuation
  }
}
```

### Responsibilities

1. **Manage conversation state** - `messages[]` accumulates history
2. **Coordinate API calls** - Build requests, process responses
3. **Trigger tool execution** - Call `StreamingToolExecutor`
4. **Handle error recovery** - prompt-too-long, max-output-tokens
5. **Control loop continuation** - Determine if next round needed

### Key Concepts

- **State struct** - Mutable state across iterations
- **turnCount** - Turn counter
- **transition** - Continuation reason marker

---

## Layer 02: TOOL DISPATCH — Tool Dispatch

### What It Is

A plugin-style **tool registration and dispatch system**, located in `src/tools.ts` and `src/Tool.ts`. ~50 built-in tools, registered and invoked through a unified `Tool` interface.

### Core Code

```typescript
// src/tools.ts:193-251
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool, GlobTool, GrepTool,
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool,
    // ... more conditional tools
  ]
}
```

### Tool Interface

```typescript
// src/Tool.ts:362-400
export type Tool<Input, Output, P> = {
  name: string
  aliases?: string[]
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  readonly inputSchema: Input
  isConcurrencySafe?(input: Input): boolean
  isReadOnly?(input: Input): boolean
}
```

### Tool Categories

| Category | Tools | Description |
|---------|-------|-------------|
| **File Operations** | Read, Write, Edit, Glob, Grep | File I/O and search |
| **Execution** | Bash, PowerShell | Shell command execution |
| **Web** | WebFetch, WebSearch | Network access |
| **Tasks** | TaskCreate, TaskUpdate, TaskList | Task management |
| **Agent** | Agent, Fork | Subagent creation |
| **Other** | TodoWrite, NotebookEdit, MCP, Skill | Various features |

### Responsibilities

1. **Tool discovery** - Register all available tools
2. **Tool execution** - Unified calling interface
3. **Concurrency control** - `isConcurrencySafe` check
4. **Permission checking** - `canUseTool` decision

---

## Layer 03: PLANNING — Plan Mode

### What It Is

Plan-first execution mode, located in `src/tools/EnterPlanModeTool.ts` and related files. The model formulates a plan first, then executes after user confirmation.

### Core Tools

| Tool | File | Description |
|------|------|-------------|
| `EnterPlanModeTool` | `src/tools/EnterPlanModeTool/` | Enter plan mode |
| `ExitPlanModeTool` | `src/tools/ExitPlanModeTool/` | Exit plan mode |
| `TodoWriteTool` | `src/tools/TodoWriteTool/` | Create todos |
| `ExitPlanModeV2Tool` | `src/tools/ExitPlanModeTool/` | V2 exit |

### Workflow

```
User: "Refactor the login module"
    ↓
/plan or auto-enter plan mode
    ↓
Model formulates plan (e.g., TODO list)
    ↓
User confirms plan
    ↓
ExitPlanModeTool
    ↓
Model executes plan
```

### Key Features

- **Plan file** - Plan content can be persisted
- **TODO tracking** - `TodoWriteTool` tracks progress
- **User control** - User confirms before each step (or auto mode)

### Source Index

- Plan mode entry: `src/tools/EnterPlanModeTool/`
- Plan mode exit: `src/tools/ExitPlanModeTool/`
- TODO tool: `src/tools/TodoWriteTool/constants.ts`

---

## Layer 04: SUB-AGENTS — Sub-Agents

### What It Is

**Fork agent** system, located in `src/tools/AgentTool/` and `src/utils/forkedAgent.ts`. Allows the model to create subtasks executing in isolated contexts.

### Core Concept

```typescript
// Enable fork agents
if (isForkSubagentEnabled()) {
  // AgentTool without subagent_type creates a fork
}
```

### Fork vs Subagent

| Feature | Fork | Subagent |
|---------|------|----------|
| Context | **Isolated**, no pollution to main conversation | Shared partial context |
| Location | **Background** | Can be foreground |
| Communication | Results can be retrieved | Real-time interaction |
| Use case | Research/multi-step implementation | Parallel task execution |

### Workflow

```
Model decides: "This task needs deep research"
    ↓
AgentTool (fork=true)
    ↓
Create isolated process/thread
    ↓
Assign isolated ToolUseContext
    ↓
Execute in background
    ↓
Inject results into main conversation on completion
```

### Key Files

- `src/tools/AgentTool/AgentTool.tsx` - Agent tool implementation
- `src/tools/AgentTool/forkSubagent.ts` - Fork logic
- `src/tools/AgentTool/runAgent.ts` - Agent execution
- `src/utils/forkedAgent.ts` - Fork agent core

---

## Layer 05: KNOWLEDGE ON DEMAND — On-Demand Knowledge

### What It Is

**Lazy-loaded** knowledge system, including Skills and Memory, injected into context only when needed.

### Two Subsystems

#### 5.1 Skills

```
src/tools/SkillTool/
src/commands.js (skill command definitions)
```

Users configure slash commands (`/commit`, `/review`) to trigger skills.

#### 5.2 Memory

```
src/memdir/memdir.ts      - loadMemoryPrompt()
src/memdir/memoryTypes.ts  - Four-type taxonomy
src/utils/claudemd.ts      - CLAUDE.md loading
```

### Memory Types

| Type | Description | Scope |
|------|-------------|-------|
| `user` | User's role, goals, preferences | Always private |
| `feedback` | User guidance (avoid/repeat) | Private or team |
| `project` | Project context (ongoing work) | Private or team |
| `reference` | External system pointers | Usually team |

### Loading Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    System Prompt (every turn)               │
│  # Memory Section - Index content                         │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    Attachments (on-demand)                  │
│  nested_memory - Loaded when relevant file accessed        │
└─────────────────────────────────────────────────────────────┘
```

### Key Files

- `src/constants/prompts.ts:495` - Memory section integration
- `src/memdir/memdir.ts:419` - `loadMemoryPrompt()`
- `src/utils/claudemd.ts:790` - `getMemoryFiles()`

---

## Layer 06: CONTEXT COMPRESSION — Context Compression

### What It Is

Three-layer compression system to prevent context overflow. Located in `src/services/compact/`.

### Three-Layer Strategy

| Layer | Name | Feature Flag | Description |
|-------|------|-------------|-------------|
| **Layer 1** | snipCompact | `HISTORY_SNIP` | Trim middle **historical** messages |
| **Layer 2** | microcompact | `CACHED_MICROCOMPACT` | Light compression within a turn |
| **Layer 3** | autoCompact | (always on) | **Summarize** when approaching limit |

### How It Works

```
Context growth:
[msg1][msg2][msg3]...[msgN][recent]
    ↓
snipCompact: Remove middle, keep head and tail
[msg1]...[recent]

    ↓
microcompact: Tool result caching, lightweight

    ↓
autoCompact: Summarize when approaching limit
[summary][recent]
```

### Trigger

```typescript
// src/query.ts - Check before API call
if (needsCompact(autoCompactTracking)) {
  autoCompactTracking = autoCompactIfNeeded(messages, tracking)
}
```

### Key Files

- `src/services/compact/autoCompact.ts` - Auto compression
- `src/services/compact/snipCompact.ts` - History trimming
- `src/services/compact/compact.ts` - Compression core

---

## Layer 07: PERSISTENT TASKS — Persistent Tasks

### What It Is

**File-based** task system where task state persists to disk, surviving conversation ends. Located in `src/tools/Task*.ts`.

### Four Core Tools

| Tool | Description | State Location |
|------|-------------|---------------|
| `TaskCreateTool` | Create task | `~/.claude/tasks/` |
| `TaskUpdateTool` | Update task status | File system |
| `TaskGetTool` | Get task details | Persisted |
| `TaskListTool` | List all tasks | - |

### Task State

```typescript
type TaskState = {
  id: string
  status: 'pending' | 'in_progress' | 'completed'
  title: string
  description?: string
  createdAt: number
  updatedAt: number
}
```

### Workflow

```
TaskCreateTool(title="Fix login bug")
    ↓
Write to ~/.claude/tasks/<id>.json
    ↓
TaskUpdateTool(id, status="in_progress")
    ↓
Update file
    ↓
Complete
```

### Difference from TODO

| Feature | Task | TODO |
|---------|------|------|
| Persisted | ✅ Disk | ❌ Memory only |
| Cross-session | ✅ Yes | ❌ No |
| State machine | Full state | Simple checkbox |

### Key Files

- `src/tools/TaskCreateTool/`
- `src/tools/TaskUpdateTool/`
- `src/tools/TaskGetTool/`
- `src/tools/TaskListTool/`

---

## Layer 08: BACKGROUND TASKS — Background Tasks

### What It Is

**Async daemon** operations. The model can run slow tasks in background without blocking the main conversation. Located in `src/tasks/`.

### Two Types

| Type | File | Description |
|------|------|-------------|
| `DreamTask` | `src/tasks/DreamTask/` | Background thinking/research |
| `LocalShellTask` | `src/tasks/LocalShellTask/` | Local shell in background |

### Workflow

```
Slow operation called in main loop
    ↓
Create background task
    ↓
Return immediately (non-blocking)
    ↓
Execute in background
    ↓
Inject notification on completion
```

### DreamTask Details

```typescript
// Long-duration research task
// Model initiates research, main loop continues responding to user
// Notified via notifyCommandLifecycle on completion
```

### LocalShellTask Details

```typescript
// Long-running shell commands
// e.g., npm install, docker build
// Run in background, real-time output viewable in UI
```

### Key Files

- `src/tasks/DreamTask/DreamTask.ts`
- `src/tasks/LocalShellTask/LocalShellTask.ts`
- `src/utils/task/framework.ts` - Task framework

---

## Layer 09: AGENT TEAMS — Multi-Agent Teams

### What It Is

**Multi-agent collaboration** system where multiple agents work together. Located in `src/tools/TeamCreateTool/` and `src/tasks/InProcessTeammateTask/`.

### Core Components

| Component | Description |
|-----------|-------------|
| `TeamCreateTool` | Create agent team |
| `TeamDeleteTool` | Delete team |
| `InProcessTeammateTask` | In-process teammate task |

### Team Architecture

```
TeamCreateTool(name="frontend-refactor")
    ↓
Create team + add team leader
    ↓
Leader decomposes tasks
    ↓
Team members (Teammates) execute subtasks in parallel
    ↓
Results aggregated
```

### In-Process vs Out-of-Process

| Mode | Communication | Use Case |
|------|---------------|----------|
| **In-Process** | Shared memory | High-frequency communication |
| **Remote** | IPC/RPC | Tasks requiring isolation |

### Key Files

- `src/tools/TeamCreateTool/TeamCreateTool.tsx`
- `src/tools/TeamDeleteTool/TeamDeleteTool.tsx`
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.ts`

---

## Layer 10: TEAM PROTOCOLS — Team Protocols

### What It Is

**Inter-agent messaging** protocols defining how agents communicate. Located in `src/tools/SendMessageTool/`.

### SendMessageTool

```typescript
// Send message between agents
SendMessageTool({
  teamId: "team-123",
  teammateId: "teammate-456",
  message: "Task completed, check results",
  type: "request" | "response" | "notification"
})
```

### Message Types

| Type | Description | Behavior |
|------|-------------|----------|
| `request` | Request | Wait for response |
| `response` | Response | Reply to request |
| `notification` | Notification | No response needed |

### Protocol Flow

```
Agent A                       Agent B
  │                              │
  │──── SendMessageTool ─────────→│
  │    type: "request"            │
  │                              │
  │←─── SendMessageTool ──────────│
  │    type: "response"          │
```

### Key Files

- `src/tools/SendMessageTool/SendMessageTool.tsx`
- `src/tools/SendMessageTool/prompt.ts` - Message prompt

---

## Layer 11: AUTONOMOUS AGENTS — Autonomous Agents

### What It Is

Coordination mode where agents **automatically claim tasks** during idle cycles. Located in `src/coordinator/coordinatorMode.ts` (Feature Flag: `COORDINATOR_MODE`).

### KAIROS Mode

```typescript
// KAIROS - Autonomous agent mode
if (feature('KAIROS') && getKairosActive()) {
  // Model checks task queue during idle
  // Automatically claims and executes tasks
}
```

### Workflow

```
User away / Model idle
    ↓
Coordinator detects idle
    ↓
Check task queue
    ↓
Model automatically claims task
    ↓
Execute (without user interaction)
    ↓
Notify user on completion
```

### Key Features

- **Heartbeat mechanism** - `<tick>` tags keep agent alive
- **Push notifications** - Notify on task state changes
- **Autonomous decisions** - Act without user指令

### Feature Flags

| Flag | Description |
|------|-------------|
| `KAIROS` | Autonomous agent main switch |
| `KAIROS_BRIEF` | Brief mode |
| `PROACTIVE` | Proactive mode |

### Key Files

- `src/coordinator/coordinatorMode.ts` - Coordinator mode
- `src/proactive/index.js` - Proactive mode (missing, one of 108 modules)

---

## Layer 12: WORKTREE ISOLATION — Worktree Isolation

### What It Is

**Git Worktree** isolation where each agent works in an isolated directory. Located in `src/tools/EnterWorktreeTool/`.

### Core Tools

| Tool | Description |
|------|-------------|
| `EnterWorktreeTool` | Enter/create worktree |
| `ExitWorktreeTool` | Exit worktree |

### How It Works

```
Git Worktree: Work in different directories of the same repo
    ↓
main worktree: ~/project/
    ↓
feature worktree: ~/project-feature-a/
    ↓
Both directories can work on different branches simultaneously
```

### Use Cases

```
Main development                 Branch isolation
    │                                │
main worktree                  feature worktree
    │                                │
Fix urgent bug ←─────────────── Long refactoring task
    │                                │
No interruption              No pollution to main
```

### Key Files

- `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
- `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`
- `src/utils/worktree.ts` - Worktree utility functions

---

## Summary: Layer Relationships

```
Layer 01: THE LOOP
    │
    ├── Layer 02: TOOL DISPATCH
    │       │
    │       ├── Layer 03: PLANNING (uses Tool interface)
    │       │
    │       ├── Layer 04: SUB-AGENTS (creates new Agent)
    │       │
    │       └── Layer 05: KNOWLEDGE ON DEMAND (tool-triggered loading)
    │
    ├── Layer 06: CONTEXT COMPRESSION (protects Loop context)
    │
    └── Layer 07: PERSISTENT TASKS (task persistence)
            │
            ├── Layer 08: BACKGROUND TASKS (async execution)
            │
            └── Layer 09: AGENT TEAMS (multi-agent collaboration)
                    │
                    ├── Layer 10: TEAM PROTOCOLS (agent communication)
                    │
                    └── Layer 11: AUTONOMOUS AGENTS (autonomous execution)
                            │
                            └── Layer 12: WORKTREE ISOLATION (isolated environment)
```

---

## Design Principles

1. **Each layer evolves independently** - Doesn't affect other layers
2. **Incremental capability addition** - Doesn't modify core Loop
3. **Uniform interface** - Layer 02's Tool interface is foundation
4. **On-demand activation** - Feature Flags control feature visibility
5. **Error isolation** - Each layer handles errors independently

---

*Based on Claude Code v2.1.88 source code analysis*
