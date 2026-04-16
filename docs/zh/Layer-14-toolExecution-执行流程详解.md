# toolExecution.ts 执行流程详解

> 工具执行统一处理层源码分析 · 2026-04-16

---

## 目录

1. [概述](#1-概述)
2. [核心函数](#2-核心函数)
3. [执行流程详解](#3-执行流程详解)
4. [错误处理](#4-错误处理)
5. [权限系统集成](#5-权限系统集成)
6. [Hook 系统集成](#6-hook-系统集成)
7. [遥测与日志](#7-遥测与日志)
8. [关键代码位置索引](#8-关键代码位置索引)

---

## 1. 概述

**文件**: `src/services/tools/toolExecution.ts`

**核心职责**: 所有工具（40+个）执行的统一入口、错误处理、权限控制、结果映射。

### 导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `runToolUse` | 337 | 主入口，AsyncGenerator，yield 消息更新 |
| `checkPermissionsAndCallTool` | 599 | 内部函数，权限检查 + 工具执行 |
| `streamedCheckPermissionsAndCallTool` | 492 | 包装为 AsyncIterable，支持进度事件 |
| `classifyToolError` | 150 | 错误分类（遥测安全） |
| `buildSchemaNotSentHint` | 578 | Zod 验证失败时提供提示 |

---

## 2. 核心函数

### 2.1 `runToolUse` 主入口

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void>
```

**职责**:
1. 根据 `toolUse.name` 查找工具（支持别名回退）
2. 处理工具不存在的情况（行 369-410）
3. 处理 AbortController 已中止的情况（行 415-452）
4. 调用 `streamedCheckPermissionsAndCallTool` 执行工具

**消息类型**: `MessageUpdateLazy = { message?: Message; contextModifier? }`

### 2.2 `checkPermissionsAndCallTool` 内部执行

```typescript
async function checkPermissionsAndCallTool(
  tool: Tool,
  toolUseID: string,
  input: { [key: string]: boolean | string | number },
  // ...
): Promise<MessageUpdateLazy[]>
```

这是真正的执行逻辑，返回 `MessageUpdateLazy[]`。

---

## 3. 执行流程详解

### 3.1 完整流程图

```
runToolUse() 入口
    │
    ├─ 查找工具（findToolByName）
    │      │
    │      └─ 工具不存在 → 返回 is_error: true (行 369-410)
    │
    ├─ 检查 AbortController
    │      │
    │      └─ 已中止 → 返回 CANCEL_MESSAGE (行 415-452)
    │
    └─ streamedCheckPermissionsAndCallTool() (行 455-468)
           │
           ▼
    checkPermissionsAndCallTool() (行 599+)
           │
           ├─ 1. Zod 输入验证（行 614-680）
           │      └─ 失败 → 返回 InputValidationError
           │
           ├─ 2. Tool.validateInput() 验证（行 682-733）
           │      └─ 失败 → 返回 tool validation error
           │
           ├─ 3. PreToolUse Hooks 执行（行 800-862）
           │      │
           ├─ 4. 权限检查 resolveHookPermissionDecision（行 921-929）
           │      │
           │      └─ 权限拒绝 → 执行 PermissionDenied Hooks（行 995-1103）
           │
           ├─ 5. 工具执行 tool.call()（行 1207-1222）
           │      │
           │      ├─ 成功 → mapToolResultToToolResultBlockParam（行 1292-1295）
           │      │      │
           │      │      └─ PostToolUse Hooks（行 1397+）
           │      │
           │      └─ 抛出异常 → catch 块处理（行 1589+）
           │
           └─ 返回结果消息
```

### 3.2 步骤详解

#### 步骤 1: Zod 输入验证（行 614-680）

```typescript
const parsedInput = tool.inputSchema.safeParse(input)
if (!parsedInput.success) {
  let errorContent = formatZodValidationError(tool.name, parsedInput.error)
  // 如果是 deferred tool 未在 schema 中，添加 buildSchemaNotSentHint
  return [{
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: `<tool_use_error>InputValidationError: ${errorContent}</tool_use_error>`,
        is_error: true,
        tool_use_id: toolUseID,
      }]
    })
  }]
}
```

**关键点**:
- `formatZodValidationError()` 生成人类可读错误（区分缺失参数、类型不匹配等）
- `buildSchemaNotSentHint()` 帮助模型理解 deferred tool 需要先加载

#### 步骤 2: Tool.validateInput() 验证（行 682-733）

```typescript
const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)
if (isValidCall?.result === false) {
  return [{
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: `<tool_use_error>${isValidCall.message}</tool_use_error>`,
        is_error: true,
        tool_use_id: toolUseID,
      }]
    })
  }]
}
```

**关键点**:
- 这是 Tool 自己的验证逻辑（如检查文件是否存在）
- 可以返回 `errorCode` 用于遥测分类

#### 步骤 3: PreToolUse Hooks（行 800-862）

```typescript
for await (const result of runPreToolUseHooks(...)) {
  switch (result.type) {
    case 'message':
      // 收集 hook 发出的消息
      resultingMessages.push(result.message)
      break
    case 'hookPermissionResult':
      hookPermissionResult = result.hookPermissionResult
      break
    case 'hookUpdatedInput':
      processedInput = result.updatedInput
      break
    case 'preventContinuation':
      shouldPreventContinuation = true
      break
    case 'stop':
      // Hook 决定停止执行
      return [{
        message: createUserMessage({
          content: [createToolResultStopMessage(toolUseID)]
        })
      }]
  }
}
```

**关键点**:
- Hook 可以修改输入 (`hookUpdatedInput`)
- Hook 可以发出权限决策 (`hookPermissionResult`)
- Hook 可以阻止执行 (`stop`)

#### 步骤 4: 权限检查（行 921-1103）

```typescript
const resolved = await resolveHookPermissionDecision(
  hookPermissionResult,
  tool,
  processedInput,
  toolUseContext,
  canUseTool,
  assistantMessage,
  toolUseID,
)
const permissionDecision = resolved.decision
processedInput = resolved.input

if (permissionDecision.behavior !== 'allow') {
  // 处理权限拒绝
  return [{
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: errorMessage,
        is_error: true,
        tool_use_id: toolUseID,
      }]
    })
  }]
}
```

#### 步骤 5: 工具执行（行 1207-1222）

```typescript
try {
  const result = await tool.call(
    callInput,
    { ...toolUseContext, toolUseId: toolUseID, userModified: ... },
    canUseTool,
    assistantMessage,
    progress => { onToolProgress(progress) },
  )
  // 成功处理...
} catch (error) {
  // 异常处理（见错误处理章节）
}
```

---

## 4. 错误处理

### 4.1 错误类型与返回位置

| 错误类型 | 返回位置 | is_error |
|---------|---------|----------|
| 工具不存在 | runToolUse:369-410 | true |
| AbortController 中止 | runToolUse:415-452 | false (CANCEL_MESSAGE) |
| Zod 输入验证失败 | checkPermissionsAndCallTool:664-679 | true |
| Tool.validateInput 失败 | checkPermissionsAndCallTool:717-732 | true |
| 权限拒绝 | checkPermissionsAndCallTool:1064-1071 | true |
| PreToolUse Hook stop | checkPermissionsAndCallTool:853-860 | false (STOP_MESSAGE) |
| 工具执行异常 | checkPermissionsAndCallTool:1589+ | true |

### 4.2 异常捕获（行 1589-1745）

```typescript
try {
  const result = await tool.call(...)
} catch (error) {
  // 1. McpAuthError → 更新 MCP needs-auth 状态
  if (error instanceof McpAuthError) {
    toolUseContext.setAppState(prevState => {
      // 更新 mcp.clients 为 needs-auth
    })
  }

  // 2. 非 AbortError → 记录日志和遥测
  if (!(error instanceof AbortError)) {
    logForDebugging(...)
    if (!(error instanceof ShellError)) logError(error)
    logEvent('tengu_tool_use_error', { ... })
  }

  // 3. 运行 PostToolUseFailure hooks
  for await (const hookResult of runPostToolUseFailureHooks(...)) {
    hookMessages.push(hookResult)
  }

  // 4. 返回错误消息
  return [{
    message: createUserMessage({
      content: [{
        type: 'tool_result',
        content: formatError(error),
        is_error: true,
        tool_use_id: toolUseID,
      }],
      toolUseResult: `Error: ${content}`,
    }),
    ...hookMessages,
  }]
}
```

### 4.3 formatError 格式化（src/utils/toolErrors.ts）

| 错误类型 | 格式化结果 |
|---------|-----------|
| `AbortError` | `INTERRUPT_MESSAGE_FOR_TOOL_USE` |
| `ShellError` | 退出码 + stderr + stdout |
| 其他 | `error.message`（截断至 10k 字符） |

---

## 5. 权限系统集成

### 5.1 权限决策流程

```
resolveHookPermissionDecision() (行 921)
    │
    ├─ 有 hookPermissionResult → 使用它
    │
    ├─ 无 → 调用 canUseTool() 询问用户
    │
    └─ 返回 PermissionResult { behavior: 'allow' | 'deny' | 'ask', ... }
```

### 5.2 权限拒绝处理（行 995-1103）

```typescript
if (permissionDecision.behavior !== 'allow') {
  // 1. 记录日志
  logEvent('tengu_tool_use_can_use_tool_rejected', { ... })

  // 2. 构建错误消息
  let errorMessage = permissionDecision.message
  if (shouldPreventContinuation && !errorMessage) {
    errorMessage = `Execution stopped by PreToolUse hook${stopReason ? `: ${stopReason}` : ''}`
  }

  // 3. 创建 tool_result 消息
  const messageContent: ContentBlockParam[] = [{
    type: 'tool_result',
    content: errorMessage,
    is_error: true,
    tool_use_id: toolUseID,
  }]

  // 4. 添加图片 blocks（如果有）
  if (rejectContentBlocks?.length) {
    messageContent.push(...rejectContentBlocks)
  }

  // 5. 执行 PermissionDenied hooks（仅 auto-mode classifier 拒绝）
  if (feature('TRANSCRIPT_CLASSIFIER') &&
      permissionDecision.decisionReason?.type === 'classifier' &&
      permissionDecision.decisionReason.classifier === 'auto-mode') {
    // 如果 hook 返回 retry: true，添加重试提示
  }
}
```

---

## 6. Hook 系统集成

### 6.1 PreToolUse Hooks（行 800-862）

```typescript
for await (const result of runPreToolUseHooks(
  toolUseContext,
  tool,
  processedInput,
  toolUseID,
  assistantMessage.message.id,
  requestId,
  mcpServerType,
  mcpServerBaseUrl,
)) {
  // 处理各种 result.type
}
```

**返回类型**:
- `message`: Hook 发出的消息（进度、附件等）
- `hookPermissionResult`: Hook 的权限决策
- `hookUpdatedInput`: Hook 修改后的输入
- `preventContinuation`: 阻止继续执行
- `stopReason`: 停止原因
- `additionalContext`: 额外上下文
- `stop`: 完全停止工具执行

### 6.2 PostToolUse Hooks（行 1397+）

```typescript
// 成功时
const hookResults = []
for await (const result of runPostToolUseHooks(...)) {
  hookResults.push(result)
}

// 失败时
for await (const result of runPostToolUseFailureHooks(...)) {
  hookMessages.push(result)
}
```

---

## 7. 遥测与日志

### 7.1 事件列表

| 事件 | 触发条件 | 行号 |
|------|---------|------|
| `tengu_tool_use_error` | 任何工具错误 | 372, 635, 691, 1643 |
| `tengu_tool_use_cancelled` | AbortController 中止 | 416 |
| `tengu_tool_use_success` | 工具执行成功 | 1331 |
| `tengu_tool_use_can_use_tool_rejected` | 权限拒绝 | 1001 |
| `tengu_tool_use_can_use_tool_allowed` | 权限允许 | 1105 |
| `tengu_tool_use_progress` | 工具进度 | 522 |
| `tengu_deferred_tool_schema_not_sent` | deferred tool 未在 schema | 625 |

### 7.2 错误分类（classifyToolError, 行 150-171）

```typescript
export function classifyToolError(error: unknown): string {
  if (error instanceof TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS) {
    return error.telemetryMessage.slice(0, 200)
  }
  if (error instanceof Error) {
    // Node.js fs 错误
    const errnoCode = getErrnoCode(error)
    if (errnoCode) return `Error:${errnoCode}`

    // 有 .name 属性
    if (error.name && error.name !== 'Error' && error.name.length > 3) {
      return error.name.slice(0, 60)
    }
  }
  return 'Error'
}
```

---

## 8. 关键代码位置索引

| 功能 | 行号 |
|------|------|
| runToolUse 主入口 | 337-490 |
| 工具不存在处理 | 369-410 |
| AbortController 检查 | 415-452 |
| streamedCheckPermissionsAndCallTool | 492-570 |
| checkPermissionsAndCallTool | 599-1745 |
| Zod 输入验证 | 614-680 |
| Tool.validateInput 调用 | 682-733 |
| PreToolUse Hooks | 800-862 |
| 权限检查 | 921-1103 |
| 权限拒绝处理 | 995-1103 |
| 工具执行 try | 1206-1222 |
| mapToolResultToToolResultBlockParam | 1292-1295 |
| 成功日志 | 1331-1395 |
| PostToolUse Hooks | 1397+ |
| 异常捕获 catch | 1589-1745 |
| formatError | utils/toolErrors.ts:5-22 |
| classifyToolError | 150-171 |

---

## 附录: 消息流完整图

```
API 返回 tool_use 块
    │
    ▼
StreamingToolExecutor.executeTool() (StreamingToolExecutor.ts:265)
    │
    ▼
runToolUse() (toolExecution.ts:337)
    │
    ├─ 工具不存在 → createUserMessage({is_error:true}) → return
    ├─ AbortController 中止 → CANCEL_MESSAGE → return
    │
    └─ checkPermissionsAndCallTool() (行 599)
           │
           ├─ Zod 验证失败 → return [{message: is_error:true}]
           ├─ validateInput 失败 → return [{message: is_error:true}]
           ├─ PreToolUse Hooks
           ├─ 权限检查
           │     └─ 拒绝 → return [{message: is_error:true}]
           │
           └─ tool.call() (行 1207)
                 │
                 ├─ 成功 → mapToolResultToToolResultBlockParam
                 │              │
                 │              ▼
                 │         StreamingToolExecutor 收集消息
                 │
                 └─ 抛出异常 → catch (行 1589)
                                  │
                                  ▼
                             return [{message: is_error:true}]

    │
    ▼
StreamingToolExecutor.getCompletedResults() (行 412-438)
    │
    ▼
query.ts 收集 toolResults (行 851-860)
    │
    ▼
下一次 API 调用: messages: [...messagesForQuery, ...assistantMessages, ...toolResults]
```
