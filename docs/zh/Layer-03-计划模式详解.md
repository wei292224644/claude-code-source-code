# Layer 03: PLANNING — 计划模式

## 概述

**PLANNING** 是 Plan-first 执行模式。模型先制定计划，用户确认后再执行。位于 `src/tools/EnterPlanModeTool/`、`src/tools/ExitPlanModeTool/` 和 `src/tools/TodoWriteTool/`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 进入计划模式 | `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts` |
| 进入计划模式提示词 | `src/tools/EnterPlanModeTool/prompt.ts` |
| 退出计划模式 V2 | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` |
| 退出计划模式提示词 | `src/tools/ExitPlanModeTool/prompt.ts` |
| TodoWrite 工具 | `src/tools/TodoWriteTool/TodoWriteTool.ts` |
| 计划文件工具 | `src/tools/TodoWriteTool/constants.ts` |
| 计划持久化 | `src/utils/plans.ts` |
| 权限模式转换 | `src/utils/permissions/permissionSetup.ts` |

---

## 计划模式工具

### 1. EnterPlanModeTool

```typescript
// src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:36-126
export const EnterPlanModeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_PLAN_MODE_TOOL_NAME,
  searchHint: 'switch to plan mode to design an approach before coding',
  shouldDefer: true,  // 延迟加载
  isConcurrencySafe() { return true },
  isReadOnly() { return true },

  async call(_input, context) {
    // 1. 检查是否在 agent 上下文中
    if (context.agentId) {
      throw new Error('EnterPlanMode tool cannot be used in agent contexts')
    }

    // 2. 处理计划模式转换
    const appState = context.getAppState()
    handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

    // 3. 更新权限模式为 'plan'
    context.setAppState(prev => ({
      ...prev,
      toolPermissionContext: applyPermissionUpdate(
        prepareContextForPlanMode(prev.toolPermissionContext),
        { type: 'setMode', mode: 'plan', destination: 'session' },
      ),
    }))

    return {
      data: {
        message: 'Entered plan mode. You should now focus on exploring...',
      },
    }
  },
})
```

### 2. ExitPlanModeV2Tool

```typescript
// src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:147-493
export const ExitPlanModeV2Tool: Tool<InputSchema, Output> = buildTool({
  name: EXIT_PLAN_MODE_V2_TOOL_NAME,
  shouldDefer: true,

  async validateInput(_input, { getAppState, options }) {
    // 仅在计划模式中可用
    const mode = getAppState().toolPermissionContext.mode
    if (mode !== 'plan') {
      return {
        result: false,
        message: 'You are not in plan mode...',
        errorCode: 1,
      }
    }
    return { result: true }
  },

  async checkPermissions(input, context) {
    if (isTeammate()) {
      // 队友：绕过权限 UI
      return { behavior: 'allow', updatedInput: input }
    }
    // 非队友：需要用户确认
    return { behavior: 'ask', message: 'Exit plan mode?', updatedInput: input }
  },

  async call(input, context) {
    // 1. 获取计划内容
    const filePath = getPlanFilePath(context.agentId)
    const plan = input.plan ?? getPlan(context.agentId)

    // 2. 队友需要队长批准
    if (isTeammate() && isPlanModeRequired()) {
      // 发送批准请求到队长
      await writeToMailbox('team-lead', {
        from: agentName,
        planContent: plan,
        requestId,
      })
      return { data: { plan, awaitingLeaderApproval: true, requestId } }
    }

    // 3. 恢复权限模式
    context.setAppState(prev => {
      setHasExitedPlanMode(true)
      setNeedsPlanModeExitAttachment(true)
      let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'
      // 处理 auto mode gate fallback
      return {
        ...prev,
        toolPermissionContext: {
          ...baseContext,
          mode: restoreMode,
          prePlanMode: undefined,
        },
      }
    })

    return { data: { plan, isAgent, filePath, hasTaskTool } }
  },
})
```

### 3. TodoWriteTool

```typescript
// src/tools/TodoWriteTool/TodoWriteTool.ts:31-115
export const TodoWriteTool = buildTool({
  name: TODO_WRITE_TOOL_NAME,
  strict: true,
  shouldDefer: true,

  async call({ todos }, context) {
    const todoKey = context.agentId ?? getSessionId()
    const oldTodos = appState.todos[todoKey] ?? []

    // 更新 todo 列表
    context.setAppState(prev => ({
      ...prev,
      todos: { ...prev.todos, [todoKey]: newTodos },
    }))

    return {
      data: { oldTodos, newTodos: todos, verificationNudgeNeeded },
    }
  },
})
```

---

## 计划模式工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    计划模式工作流程                                   │
│                                                                     │
│  用户: "重构登录模块"                                                 │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 方案1: 模型主动调用 EnterPlanModeTool                        │   │
│  │ 方案2: 用户输入 /plan                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  handlePlanModeTransition()                                         │
│       ↓                                                             │
│  权限模式: default → plan                                           │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 计划模式: 探索代码库，设计实现方案                            │   │
│  │                                                             │   │
│  │ 1. 使用 Glob/Grep/Read 探索代码                             │   │
│  │ 2. 理解现有模式                                             │   │
│  │ 3. 设计实现方案                                             │   │
│  │ 4. 使用 AskUserQuestion 澄清方法                             │   │
│  │ 5. 使用 TodoWriteTool 创建任务清单                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  模型: "我已经完成计划，请审批"                                       │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ ExitPlanModeV2Tool                                          │   │
│  │                                                             │   │
│  │ 1. validateInput() - 验证在计划模式中                       │   │
│  │ 2. checkPermissions() - 请求用户确认                         │   │
│  │ 3. call() - 恢复权限模式                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       ↓                                                             │
│  权限模式: plan → default (或 auto)                                 │
│       ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 执行模式: 实施计划                                           │   │
│  │                                                             │   │
│  │ 1. 按照计划执行任务                                         │   │
│  │ 2. 使用 TodoWriteTool 追踪进度                             │   │
│  │ 3. 验证实施结果                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 进入计划模式决策

### EnterPlanModeTool 提示词中的决策指导

```typescript
// src/tools/EnterPlanModeTool/prompt.ts:26-56
// 何时使用 EnterPlanMode
// 1. **新功能实现**: 添加有意义的新功能
// 2. **多种方案**: 任务有多种解决方案
// 3. **代码修改**: 影响现有行为或结构
// 4. **架构决策**: 需要选择模式或技术
// 5. **多文件更改**: 可能涉及超过 2-3 个文件
// 6. **需求不清**: 需要探索才能理解完整范围
// 7. **用户偏好重要**: 实现有多种合理方向

// 何时不使用
// - 简单的修复（typo、明显 bug）
// - 添加单一函数（需求明确）
// - 用户给出非常具体的指令
// - 纯研究/探索任务
```

---

## 权限模式转换

### prepareContextForPlanMode

```typescript
// src/utils/permissions/permissionSetup.ts
export function prepareContextForPlanMode(context: ToolPermissionContext) {
  // 保存进入计划前的模式
  const prePlanMode = context.mode

  // 返回新的上下文，权限模式为 plan
  return {
    ...context,
    prePlanMode,
    mode: 'plan',
  }
}
```

### handlePlanModeTransition

```typescript
// src/bootstrap/state.ts
export function handlePlanModeTransition(
  fromMode: PermissionMode,
  toMode: 'plan' | 'default' | 'auto',
) {
  // 处理模式转换的副作用
  // 例如：激活分类器、记录指标等
}
```

---

## 计划持久化

### 计划文件路径

```typescript
// src/utils/plans.ts
export function getPlanFilePath(agentId?: AgentId): string | undefined {
  // 返回计划文件路径
  // 格式: ~/.claude/plans/<sessionId>/plan.md
}
```

### 计划内容读写

```typescript
// src/utils/plans.ts
export function getPlan(agentId?: AgentId): string | undefined {
  // 从磁盘读取计划内容
}

export function persistFileSnapshotIfRemote(): void {
  // 将计划快照持久化到远程
}
```

---

## 团队中的计划批准

### 队长批准流程

```
Agent (队友)                          Team Lead (队长)
    │                                     │
    │── ExitPlanModeV2Tool ──────────────→│
    │   (发送计划批准请求)                   │
    │                                     │
    │                        ┌────────────┴────────────┐
    │                        │ 审查计划                  │
    │                        │ 批准/拒绝                 │
    │                        └────────────┬────────────┘
    │                                     │
    │←── 批准/拒绝消息 ──────────────────────│
    │                                     │
    │ 继续实施（如果批准）                    │
```

---

## 关键设计

### 1. shouldDefer: true

计划模式工具使用延迟加载（`shouldDefer: true`），模型需要先通过 ToolSearch 发现工具才能调用。

### 2. 权限模式状态

```typescript
type PermissionMode = 'default' | 'auto' | 'plan' | 'bypass' | 'custom'

// 进入计划模式
prePlanMode: 'default'  // 保存原模式
mode: 'plan'            // 设置为计划模式

// 退出计划模式
mode: 'default' 或 'auto'  // 恢复到原模式
prePlanMode: undefined
```

### 3. 队友权限

```typescript
// 队友调用 ExitPlanModeTool 时
if (isTeammate()) {
  // 不需要用户交互
  // 直接发送批准请求到队长
  return { data: { awaitingLeaderApproval: true } }
}
```

---

## 与其他层的关系

```
Layer 01: THE LOOP
    │
    └── Layer 02: TOOL DISPATCH
            │
            └── Layer 03: PLANNING
                    │
                    ├── EnterPlanModeTool → 权限模式转换为 plan
                    │
                    ├── TodoWriteTool → 任务追踪
                    │
                    └── ExitPlanModeV2Tool → 权限模式恢复 + 队长批准
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| EnterPlanModeTool | `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:36-126` |
| EnterPlanModeTool 提示词 | `src/tools/EnterPlanModeTool/prompt.ts:1-170` |
| ExitPlanModeV2Tool | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:147-493` |
| ExitPlanModeV2Tool 提示词 | `src/tools/ExitPlanModeTool/prompt.ts:1-29` |
| TodoWriteTool | `src/tools/TodoWriteTool/TodoWriteTool.ts:31-115` |
| getPlanFilePath | `src/utils/plans.ts` |
| prepareContextForPlanMode | `src/utils/permissions/permissionSetup.ts` |
| handlePlanModeTransition | `src/bootstrap/state.ts` |
| ENTER_PLAN_MODE_TOOL_NAME | `src/tools/EnterPlanModeTool/constants.ts` |
| EXIT_PLAN_MODE V2 TOOL NAME | `src/tools/ExitPlanModeTool/constants.ts` |

---

*基于 Claude Code v2.1.88 源代码分析*
