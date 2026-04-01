# Claude Code 十二层 Progressive Harness 详解

## 概述

Claude Code 的 harness 采用了**分层架构**，每一层构建在前一层之上，增量添加能力而不修改核心循环。这种设计允许功能解耦、独立演进，同时保持核心逻辑的简洁。

```
┌─────────────────────────────────────────────────────────────┐
│                      Layer 12: WORKTREE ISOLATION          │
│                      (每代理目录隔离)                        │
├─────────────────────────────────────────────────────────────┤
│                      Layer 11: AUTONOMOUS AGENTS            │
│                      (空闲时自动认领任务)                    │
├─────────────────────────────────────────────────────────────┤
│                      Layer 10: TEAM PROTOCOLS               │
│                      (代理间消息传递)                        │
├─────────────────────────────────────────────────────────────┤
│                      Layer 09: AGENT TEAMS                 │
│                      (多代理协作)                            │
├─────────────────────────────────────────────────────────────┤
│                      Layer 08: BACKGROUND TASKS             │
│                      (异步守护进程)                          │
├─────────────────────────────────────────────────────────────┤
│                      Layer 07: PERSISTENT TASKS             │
│                      (基于文件的任务图)                       │
├─────────────────────────────────────────────────────────────┤
│                      Layer 06: CONTEXT COMPRESSION          │
│                      (三层压缩)                              │
├─────────────────────────────────────────────────────────────┤
│                      Layer 05: KNOWLEDGE ON DEMAND         │
│                      (懒加载技能/记忆)                       │
├─────────────────────────────────────────────────────────────┤
│                      Layer 04: SUB-AGENTS                  │
│                      (分叉代理)                              │
├─────────────────────────────────────────────────────────────┤
│                      Layer 03: PLANNING                     │
│                      (Plan-first执行)                       │
├─────────────────────────────────────────────────────────────┤
│                      Layer 02: TOOL DISPATCH                │
│                      (插件式工具注册)                       │
├─────────────────────────────────────────────────────────────┤
│                      Layer 01: THE LOOP                    │
│                      (核心while-true循环)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 01: THE LOOP — 核心循环

### 是什么

Harness 的**心脏**，位于 `src/query.ts`。一个 `while(true)` 的 async generator，持续执行 fetch-execute 循环，直到任务完成或满足退出条件。

### 核心代码

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
    // 1. Setup Phase - 初始化配置
    // 2. API Call Phase - 调用 Claude API
    // 3. Tool Execution Phase - 执行工具
    // 4. Recovery Phase - 错误恢复
    // 5. Continuation Phase - 循环继续
  }
}
```

### 职责

1. **管理对话状态** - `messages[]` 累积对话历史
2. **协调 API 调用** - 构建请求、处理响应
3. **触发工具执行** - 调用 `StreamingToolExecutor`
4. **处理错误恢复** - prompt-too-long、max-output-tokens 等
5. **控制循环继续** - 判断是否需要下一轮

### 关键概念

- **State 结构** - 跨迭代的可变状态
- **turnCount** - 轮次计数器
- **transition** - 继续原因标记

---

## Layer 02: TOOL DISPATCH — 工具调度

### 是什么

插件式的**工具注册与调度系统**，位于 `src/tools.ts` 和 `src/Tool.ts`。约 50 个内置工具，通过统一的 `Tool` 接口注册和调用。

### 核心代码

```typescript
// src/tools.ts:193-251
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool, GlobTool, GrepTool,
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool,
    // ... 更多条件工具
  ]
}
```

### 工具接口

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

### 工具分类

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | Read, Write, Edit, Glob, Grep | 文件读写搜索 |
| **执行** | Bash, PowerShell | shell 命令执行 |
| **Web** | WebFetch, WebSearch | 网络访问 |
| **任务** | TaskCreate, TaskUpdate, TaskList | 任务管理 |
| **Agent** | Agent, Fork | 子代理创建 |
| **其他** | TodoWrite, NotebookEdit, MCP, Skill | 各种功能 |

### 职责

1. **工具发现** - 注册所有可用工具
2. **工具执行** - 统一的调用接口
3. **并发控制** - `isConcurrencySafe` 判断
4. **权限检查** - `canUseTool` 决策

---

## Layer 03: PLANNING — 计划模式

### 是什么

Plan-first 执行模式，位于 `src/tools/EnterPlanModeTool.ts` 等。让模型先制定计划，用户确认后再执行。

### 核心工具

| 工具 | 文件 | 描述 |
|------|------|------|
| `EnterPlanModeTool` | `src/tools/EnterPlanModeTool/` | 进入计划模式 |
| `ExitPlanModeTool` | `src/tools/ExitPlanModeTool/` | 退出计划模式 |
| `TodoWriteTool` | `src/tools/TodoWriteTool/` | 创建待办事项 |
| `ExitPlanModeV2Tool` | `src/tools/ExitPlanModeTool/` | V2 版本退出 |

### 工作流程

```
用户: "重构登录模块"
    ↓
/plan 或 自动进入计划模式
    ↓
模型制定计划（如 TODO 列表）
    ↓
用户确认计划
    ↓
ExitPlanModeTool
    ↓
模型执行计划
```

### 关键特性

- **Plan 文件** - 计划内容可持久化
- **TODO 跟踪** - `TodoWriteTool` 追踪任务进度
- **用户控制** - 每步执行前需用户确认（或自动执行模式）

### 源码索引

- 计划模式入口: `src/tools/EnterPlanModeTool/`
- 计划模式退出: `src/tools/ExitPlanModeTool/`
- TODO 工具: `src/tools/TodoWriteTool/constants.ts`

---

## Layer 04: SUB-AGENTS — 子代理

### 是什么

**分叉代理**系统，位于 `src/tools/AgentTool/` 和 `src/utils/forkedAgent.ts`。允许模型创建子任务，在独立上下文中执行。

### 核心概念

```typescript
// 启用分叉代理
if (isForkSubagentEnabled()) {
  // AgentTool 不带 subagent_type 参数时创建分叉
}
```

### 分叉 vs 子代理

| 特性 | Fork (分叉) | Subagent (子代理) |
|------|-------------|------------------|
| 上下文 | **独立**，不污染主对话 | 共享部分上下文 |
| 位置 | **后台运行** | 可前台运行 |
| 通信 | 结果可取回 | 实时交互 |
| 用途 | 研究/多步实现 | 并行任务执行 |

### 工作流程

```
模型决定: "这个任务需要深入研究"
    ↓
AgentTool (fork=true)
    ↓
创建独立进程/线程
    ↓
分配独立 ToolUseContext
    ↓
后台执行
    ↓
完成后结果注入主对话
```

### 关键文件

- `src/tools/AgentTool/AgentTool.tsx` - Agent 工具实现
- `src/tools/AgentTool/forkSubagent.ts` - 分叉逻辑
- `src/tools/AgentTool/runAgent.ts` - 代理执行
- `src/utils/forkedAgent.ts` - 分叉代理核心

---

## Layer 05: KNOWLEDGE ON DEMAND — 按需知识

### 是什么

**懒加载**的知识系统，包括技能（Skills）和记忆（Memory），在需要时才注入上下文。

### 两大子系统

#### 5.1 Skills (技能系统)

```
src/tools/SkillTool/
src/commands.js (skill 命令定义)
```

用户可配置 slash commands (`/commit`, `/review`) 触发技能。

#### 5.2 Memory (记忆系统)

```
src/memdir/memdir.ts      - loadMemoryPrompt()
src/memdir/memoryTypes.ts  - 四类记忆定义
src/utils/claudemd.ts      - CLAUDE.md 加载
```

### 记忆类型

| 类型 | 描述 | 范围 |
|------|------|------|
| `user` | 用户角色、目标、偏好 | 始终私有 |
| `feedback` | 用户指导（避免/重复） | 私有或团队 |
| `project` | 项目上下文（进行中工作） | 私有或团队 |
| `reference` | 外部系统指针 | 通常团队 |

### 加载策略

```
┌─────────────────────────────────────────────────────────────┐
│                    System Prompt (每次)                       │
│  # Memory Section - 索引内容                               │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    Attachments (按需)                        │
│  nested_memory - 相关文件触发时加载                         │
└─────────────────────────────────────────────────────────────┘
```

### 关键文件

- `src/constants/prompts.ts:495` - 记忆章节集成
- `src/memdir/memdir.ts:419` - `loadMemoryPrompt()`
- `src/utils/claudemd.ts:790` - `getMemoryFiles()`

---

## Layer 06: CONTEXT COMPRESSION — 上下文压缩

### 是什么

三层压缩系统，防止上下文溢出。位于 `src/services/compact/`。

### 三层策略

| 层级 | 名称 | Feature Flag | 描述 |
|------|------|-------------|------|
| **Layer 1** | snipCompact | `HISTORY_SNIP` | 裁剪历史**中间**消息 |
| **Layer 2** | microcompact | `CACHED_MICROCOMPACT` | 单轮内轻量压缩 |
| **Layer 3** | autoCompact | (始终启用) | 上下文接近上限时**总结** |

### 工作原理

```
上下文增长:
[消息1][消息2][消息3]...[消息N][最近消息]
    ↓
snipCompact: 删除中间部分，保留头尾
[消息1]...[最近消息]

    ↓
microcompact: 工具结果缓存，轻量化

    ↓
autoCompact: 接近上限时总结
[总结][最近消息]
```

### 触发条件

```typescript
// src/query.ts - API 调用前检查
if (needsCompact(autoCompactTracking)) {
  autoCompactTracking = autoCompactIfNeeded(messages, tracking)
}
```

### 关键文件

- `src/services/compact/autoCompact.ts` - 自动压缩
- `src/services/compact/snipCompact.ts` - 历史裁剪
- `src/services/compact/compact.ts` - 压缩核心

---

## Layer 07: PERSISTENT TASKS — 持久化任务

### 是什么

基于**文件**的任务系统，任务状态持久化到磁盘，不受对话结束影响。位于 `src/tools/Task*.ts`。

### 四个核心工具

| 工具 | 描述 | 状态位置 |
|------|------|----------|
| `TaskCreateTool` | 创建任务 | `~/.claude/tasks/` |
| `TaskUpdateTool` | 更新任务状态 | 文件系统 |
| `TaskGetTool` | 获取任务详情 | 持久化 |
| `TaskListTool` | 列出所有任务 | - |

### 任务状态

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

### 工作流程

```
TaskCreateTool(title="修复登录bug")
    ↓
写入 ~/.claude/tasks/<id>.json
    ↓
TaskUpdateTool(id, status="in_progress")
    ↓
文件更新
    ↓
完成
```

### 与 TODO 的区别

| 特性 | Task (任务) | TODO |
|------|------------|------|
| 持久化 | ✅ 磁盘 | ❌ 仅内存 |
| 跨会话 | ✅ 是 | ❌ 否 |
| 状态机 | 完整状态 | 简单复选框 |

### 关键文件

- `src/tools/TaskCreateTool/`
- `src/tools/TaskUpdateTool/`
- `src/tools/TaskGetTool/`
- `src/tools/TaskListTool/`

---

## Layer 08: BACKGROUND TASKS — 后台任务

### 是什么

**异步守护进程**操作，模型可在后台运行慢任务，不阻塞主对话。位于 `src/tasks/`。

### 两种后台任务

| 类型 | 文件 | 描述 |
|------|------|------|
| `DreamTask` | `src/tasks/DreamTask/` | 后台思考/研究任务 |
| `LocalShellTask` | `src/tasks/LocalShellTask/` | 本地 shell 后台执行 |

### 工作流程

```
主循环中调用慢操作
    ↓
创建后台任务
    ↓
立即返回 (不阻塞)
    ↓
后台继续执行
    ↓
完成时注入通知
```

### DreamTask 详解

```typescript
// 长时间研究任务
// 模型发起研究，主循环继续响应用户
// 研究完成后通过 notifyCommandLifecycle 通知
```

### LocalShellTask 详解

```typescript
// 长时间 shell 命令
// e.g., npm install, docker build
// 后台运行，实时输出可通过 UI 查看
```

### 关键文件

- `src/tasks/DreamTask/DreamTask.ts`
- `src/tasks/LocalShellTask/LocalShellTask.ts`
- `src/utils/task/framework.ts` - 任务框架

---

## Layer 09: AGENT TEAMS — 多代理团队

### 是什么

**多代理协作**系统，多个代理协同工作。位于 `src/tools/TeamCreateTool/` 和 `src/tasks/InProcessTeammateTask/`。

### 核心组件

| 组件 | 描述 |
|------|------|
| `TeamCreateTool` | 创建代理团队 |
| `TeamDeleteTool` | 删除团队 |
| `InProcessTeammateTask` | 进程内队友任务 |

### 团队架构

```
TeamCreateTool(name="前端重构")
    ↓
创建团队 + 添加队长
    ↓
队长负责任务分解
    ↓
队员 (Teammates) 并行执行子任务
    ↓
结果汇总
```

### 进程内 vs 进程外

| 模式 | 通信 | 适用场景 |
|------|------|----------|
| **In-Process** | 共享内存 | 高频通信任务 |
| **Remote** | IPC/RPC | 需要隔离的任务 |

### 关键文件

- `src/tools/TeamCreateTool/TeamCreateTool.tsx`
- `src/tools/TeamDeleteTool/TeamDeleteTool.tsx`
- `src/tasks/InProcessTeammateTask/InProcessTeammateTask.ts`

---

## Layer 10: TEAM PROTOCOLS — 团队协议

### 是什么

**代理间消息传递**协议，定义代理如何通信。位于 `src/tools/SendMessageTool/`。

### SendMessageTool

```typescript
// 代理间发送消息
SendMessageTool({
  teamId: "team-123",
  teammateId: "teammate-456",
  message: "任务已完成，检查结果",
  type: "request" | "response" | "notification"
})
```

### 消息类型

| 类型 | 描述 | 行为 |
|------|------|------|
| `request` | 请求 | 等待响应 |
| `response` | 响应 | 回复请求 |
| `notification` | 通知 | 不需回复 |

### 协议流程

```
Agent A                       Agent B
  │                              │
  │──── SendMessageTool ─────────→│
  │    type: "request"            │
  │                              │
  │←─── SendMessageTool ──────────│
  │    type: "response"          │
```

### 关键文件

- `src/tools/SendMessageTool/SendMessageTool.tsx`
- `src/tools/SendMessageTool/prompt.ts` - 消息提示词

---

## Layer 11: AUTONOMOUS AGENTS — 自主代理

### 是什么

**空闲时自动认领任务**的协调模式。位于 `src/coordinator/coordinatorMode.ts` (Feature Flag: `COORDINATOR_MODE`)。

### KAIROS 模式

```typescript
// KAIROS - 自主代理模式
if (feature('KAIROS') && getKairosActive()) {
  // 模型在空闲时自动检查任务队列
  // 认领并执行任务
}
```

### 工作流程

```
用户离开 / 模型空闲
    ↓
Coordinator 检测空闲
    ↓
检查任务队列
    ↓
模型自动认领任务
    ↓
执行 (可在无用户交互情况下)
    ↓
完成时通知用户
```

### 关键特性

- **心跳机制** - `<tick>` 标签保持代理活跃
- **推送通知** - 任务状态变更时通知
- **自主决策** - 不需用户指令即可行动

### Feature Flags

| Flag | 描述 |
|------|------|
| `KAIROS` | 自主代理主开关 |
| `KAIROS_BRIEF` | 简短模式 |
| `PROACTIVE` | 主动模式 |

### 关键文件

- `src/coordinator/coordinatorMode.ts` - 协调者模式
- `src/proactive/index.js` - 主动模式 (缺失，108模块之一)

---

## Layer 12: WORKTREE ISOLATION — Worktree 隔离

### 是什么

**Git Worktree** 隔离，每个代理在独立目录工作。位于 `src/tools/EnterWorktreeTool/`。

### 核心工具

| 工具 | 描述 |
|------|------|
| `EnterWorktreeTool` | 进入/创建 worktree |
| `ExitWorktreeTool` | 退出 worktree |

### 工作原理

```
Git Worktree: 在同一仓库的不同目录工作
    ↓
main worktree: ~/project/
    ↓
feature worktree: ~/project-feature-a/
    ↓
两个目录可同时在不同分支工作
```

### 使用场景

```
主线开发                    分支隔离
    │                          │
main worktree              feature worktree
    │                          │
修复紧急 bug ←────────────── 长时重构任务
    │                          │
不中断                       不污染主线
```

### 关键文件

- `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
- `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`
- `src/utils/worktree.ts` - worktree 工具函数

---

## 总结：层次关系

```
Layer 01: THE LOOP
    │
    ├── Layer 02: TOOL DISPATCH
    │       │
    │       ├── Layer 03: PLANNING (使用 Tool 接口)
    │       │
    │       ├── Layer 04: SUB-AGENTS (创建新 Agent)
    │       │
    │       └── Layer 05: KNOWLEDGE ON DEMAND (工具触发加载)
    │
    ├── Layer 06: CONTEXT COMPRESSION (保护 Loop 上下文)
    │
    └── Layer 07: PERSISTENT TASKS (任务持久化)
            │
            ├── Layer 08: BACKGROUND TASKS (异步执行)
            │
            └── Layer 09: AGENT TEAMS (多代理协作)
                    │
                    ├── Layer 10: TEAM PROTOCOLS (代理通信)
                    │
                    └── Layer 11: AUTONOMOUS AGENTS (自主运行)
                            │
                            └── Layer 12: WORKTREE ISOLATION (隔离环境)
```

---

## 设计原则

1. **每层独立演进** - 不影响其他层
2. **增量添加能力** - 不修改核心 Loop
3. **统一接口** - Layer 02 的 Tool 接口是基础
4. **按需激活** - Feature Flags 控制功能可见性
5. **错误隔离** - 每层独立错误处理

---

*基于 Claude Code v2.1.88 源代码分析*
