# Harness 错误处理详解

> Claude Code 内部错误处理完整分析 · 2026-04-15

---

## 目录

1. [错误架构总览](#1-错误架构总览)
2. [错误分类体系](#2-错误分类体系)
3. [API 层错误处理](#3-api-层错误处理)
4. [Query 循环错误处理](#4-query-循环错误处理)
5. [工具执行错误处理](#5-工具执行错误处理)
6. [MCP 错误处理](#6-mcp-错误处理)
7. [权限拒绝错误](#7-权限拒绝错误)
8. [错误流图](#8-错误流图)
9. [重试策略总结](#9-重试策略总结)

---

## 1. 错误架构总览

Claude Code 采用**三层错误处理架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                     API 层 (withRetry.ts)                   │
│         HTTP 错误分类 · 重试决策 · 指数退避                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Query 循环 (query.ts)                       │
│     错误恢复策略 · 模型降级 · 上下文压缩 · 令牌升级            │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  工具执行层      │ │   MCP 层        │ │   权限系统       │
│ toolExecution   │ │   client.ts     │ │  toolHooks.ts   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

**关键文件列表：**

| 文件 | 职责 |
|------|------|
| `src/services/api/withRetry.ts` | API 重试逻辑、错误分类、延迟计算 |
| `src/services/api/errors.ts` | API 错误类型、消息生成、错误分类函数 |
| `src/query.ts` | 主循环错误处理、恢复策略 |
| `src/services/tools/toolExecution.ts` | 工具执行层错误处理 |
| `src/utils/errors.ts` | 核心错误类定义 |
| `src/utils/toolErrors.ts` | 工具错误格式化函数 |
| `src/services/mcp/client.ts` | MCP 错误类定义 |
| `src/utils/messages.ts` | 错误消息构造辅助函数 |

---

## 2. 错误分类体系

### 2.1 错误来源分类

| 分类 | 位置 | 说明 |
|------|------|------|
| **API 错误** | `services/api/` | API 调用失败（401/429/500 等） |
| **上下文过长错误** | `query.ts:1070+` | Prompt 超限（413） |
| **输出令牌超限** | `query.ts:1188+` | 模型输出超出 max_tokens |
| **工具执行错误** | `toolExecution.ts` | 工具运行失败 |
| **权限拒绝错误** | `toolExecution.ts:995+` | 工具使用被拒绝 |
| **MCP 错误** | `mcp/client.ts` | MCP 服务器相关错误 |
| **媒体大小错误** | `query.ts:1082+` | 图片/PDF 超出大小限制 |

### 2.2 API 错误分类 (`classifyAPIError`)

**文件**: `src/services/api/errors.ts:965-1161`

| 分类返回值 | 触发条件 |
|-----------|---------|
| `aborted` | `error.message === 'Request was aborted.'` |
| `api_timeout` | `APIConnectionTimeoutError` 或消息含 "timeout" |
| `repeated_529` | 连续多次 529 错误 |
| `rate_limit` | `status === 429` |
| `server_overload` | `status === 529` 或含 `"type":"overloaded_error"` |
| `prompt_too_long` | 消息含 `PROMPT_TOO_LONG_ERROR_MESSAGE` |
| `pdf_too_large` | PDF 页数超限 |
| `image_too_large` | 图片大小/尺寸超限 |
| `auth_error` | `status === 401/403` |
| `invalid_api_key` | 含 `"x-api-key"` |
| `token_revoked` | OAuth token 撤销 |
| `credit_balance_low` | 额度不足 |
| `server_error` | `status >= 500` |
| `client_error` | `status >= 400` |
| `connection_error` | `APIConnectionError`（非 SSL） |
| `ssl_cert_error` | SSL 证书错误 |
| `unknown` | 其他 |

### 2.3 SDK 错误类型 (`SDKAssistantMessageError`)

这些类型传递给 `createAssistantAPIErrorMessage` 的 `error` 字段：

| 值 | 来源 |
|----|------|
| `'rate_limit'` | 429 / 529 |
| `'authentication_failed'` | 401/403/x-api-key/OAuth revoked |
| `'billing_error'` | credit balance too low |
| `'invalid_request'` | PTL / PDF 错误 / 图片过大 / 工具错误 / 模型错误 / 400 |
| `'server_error'` | `status >= 500` |
| `'max_output_tokens'` | 输出 token 超限 |
| `'unknown'` | 其他 |

---

## 3. API 层错误处理

### 3.1 重试决策 (`shouldRetry`)

**文件**: `src/services/api/withRetry.ts:696-787`

| 条件 | 重试？ |
|------|--------|
| 429/529 (Persistent 模式) | **是** |
| 401/403 (CCR 模式) | **是** |
| 含 `"type":"overloaded_error"` | **是** |
| `x-should-retry: true` + (非订阅户 或 企业户) | **是** |
| `x-should-retry: false` + 非 Ant 5xx | **否** |
| `APIConnectionError` | **是** |
| `status === 408` | **是** |
| `status === 409` | **是** |
| `status === 429` + 非订阅户 或 企业户 | **是** |
| `status === 401` | **是**（清除 API key 缓存） |
| OAuth token revoked (403) | **是** |
| `status >= 500` | **是** |
| 其他 | **否** |

### 3.2 重试延迟计算 (`getRetryDelay`)

**文件**: `src/services/api/withRetry.ts:530-548`

- 有 `Retry-After` header → 直接解析秒数
- 无 header → 指数退避：`BASE_DELAY_MS * 2^(attempt-1)` + 25% 随机抖动
- 最大延迟：默认 32 秒

### 3.3 核心常量

**文件**: `src/services/api/withRetry.ts:52-98`

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_MAX_RETRIES` | `10` | 普通模式最大重试次数 |
| `MAX_529_RETRIES` | `3` | 529 错误最大重试次数 |
| `BASE_DELAY_MS` | `500` | 基础退避延迟 |
| `FLOOR_OUTPUT_TOKENS` | `3000` | 输出令牌下限 |
| `PERSISTENT_MAX_BACKOFF_MS` | `5分钟` | Persistent 模式最大退避 |
| `PERSISTENT_RESET_CAP_MS` | `6小时` | Persistent 模式 reset 上限 |
| `HEARTBEAT_INTERVAL_MS` | `30秒` | Persistent 模式心跳间隔 |
| `SHORT_RETRY_THRESHOLD_MS` | `20秒` | Fast mode 切换阈值 |

### 3.4 529 特殊处理（模型降级）

**文件**: `src/services/api/withRetry.ts:326-365`

触发条件：
- 连续 `MAX_529_RETRIES (3)` 次 529
- 且（非订阅户 + 非自定义 Opus 模型）**或** `FALLBACK_FOR_ALL_PRIMARY_MODELS` 已设置

处理：
1. 若指定了 `fallbackModel` → 抛出 `FallbackTriggeredError`（切换模型重新跑整个请求）
2. 否则 → 抛出 `CannotRetryError(REPEATED_529_ERROR_MESSAGE)`

### 3.5 异常类

**文件**: `src/services/api/withRetry.ts`

| 类名 | 行号 | 用途 |
|------|------|------|
| `CannotRetryError` | 144-158 | 保存原始错误和重试上下文，标识"不应再重试" |
| `FallbackTriggeredError` | 160-168 | 模型降级触发时抛出 |

### 3.6 Fast Mode 限速降级

**文件**: `src/services/api/withRetry.ts:267-305`

| 条件 | 动作 |
|------|------|
| `retryAfterMs < 20秒` | 等待后带缓存重试（同模型） |
| `retryAfterMs >= 20秒` | 进入 30 分钟 cooldown（切换标准速度模型） |
| header 含 `overage-disabled-reason` | 永久禁用 fast mode |

---

## 4. Query 循环错误处理

### 4.1 完整错误处理流程图

```
API 调用
    │
    ▼
┌─────────────────┐
│  成功？          │
└─────────────────┘
    │
  Yes│                No
    │                 │
    │                 ▼
    │         ┌───────────────────┐
    │         │ FallbackTriggered? │  ← 捕获 in query.ts:894
    │         └───────────────────┘
    │              │
    │         Yes  │              No
    │              │               │
    │              ▼               ▼
    │    清空状态              ┌─────────────┐
    │    切换模型               │ ImageSize/  │
    │    continue              │ ImageResize? │
    │    (整体重新跑)           └─────────────┘
    │                             │
    │                        Yes  │              No
    │                            │               │
    │                            ▼               ▼
    │                    直接返回            补全丢失的
    │                    image_error         tool_result
    │                                          │
    │                                          ▼
    │                               createAssistantAPIErrorMessage
    │                                          │
    │                              ┌───────────┴───────────┐
    │                              ▼                       ▼
    │                      正常退出                    退出循环
    │                      return {reason: model_error}
```

### 4.2 FallbackTriggeredError 处理

**文件**: `src/query.ts:894-950`

```typescript
} catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true
    // 清空本轮所有累积状态
    assistantMessages.length = 0
    toolResults.length = 0
    toolUseBlocks.length = 0
    needsFollowUp = false
    // 丢弃旧流式执行器并重建
    streamingToolExecutor?.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
    // 剥离签名块（保护性思考模型切换时不兼容）
    if (process.env.USER_TYPE === 'ant') {
      messagesForQuery = stripSignatureBlocks(messagesForQuery)
    }
    logEvent('tengu_model_fallback_triggered', { ... })
    yield createSystemMessage(`Switched to ${renderModelName(innerError.fallbackModel)}...`, 'warning')
    continue  // ← 整体重新跑
  }
  throw innerError
}
```

**重试策略：整体重新跑**
- 切换 `currentModel` 为 `fallbackModel`
- 清空所有 assistant messages、tool results、tool use blocks
- 创建全新的 `StreamingToolExecutor`
- 重新执行**整个**请求循环（不只调整 prompt）

### 4.3 外层 catch 块（一般错误）

**文件**: `src/query.ts:955-997`

```typescript
} catch (error) {
  logError(error)
  logEvent('tengu_query_error', { ... })

  // ImageSizeError / ImageResizeError → 直接退出
  if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
    yield createAssistantAPIErrorMessage({ content: error.message })
    return { reason: 'image_error' }
  }

  // 补全因异常中断而缺失的 tool_result 块
  yield* yieldMissingToolResultBlocks(assistantMessages, errorMessage)
  yield createAssistantAPIErrorMessage({ content: errorMessage })
  logAntError('Query error', error)
  return { reason: 'model_error', error }
}
```

### 4.4 上下文过长错误（413）

**文件**: `src/query.ts:1070-1183`

#### 识别

```typescript
const isWithheld413 =
  lastMessage?.type === 'assistant' &&
  lastMessage.isApiErrorMessage &&
  isPromptTooLongMessage(lastMessage)
```

#### 恢复策略（三层）

**第一层：Context Collapse 排水重试**（行 1085-1117）

```typescript
if (isWithheld413 && feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
  if (drained.committed > 0) {
    state = { messages: drained.messages, ... }
    continue  // ← 重新跑
  }
}
```

**第二层：Reactive Compact 压缩重试**（行 1119-1166）

```typescript
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({ ... })
  if (compacted) {
    const postCompactMessages = buildPostCompactMessages(compacted)
    for (const msg of postCompactMessages) { yield msg }
    state = { messages: postCompactMessages, hasAttemptedReactiveCompact: true, ... }
    continue  // ← 重新跑
  }
  // 恢复失败 → 暴露错误并退出
  yield lastMessage
  return { reason: 'prompt_too_long' }
}
```

**第三层：无 recovery 时直接暴露**（行 1176-1182）

```typescript
} else if (feature('CONTEXT_COLLAPSE') && isWithheld413) {
  yield lastMessage
  return { reason: 'prompt_too_long' }
}
```

### 4.5 max_output_tokens 错误

**文件**: `src/query.ts:1188-1256`

#### 识别

```typescript
function isWithheldMaxOutputTokens(msg): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

#### 恢复策略（三层）

| 层级 | 行号 | 策略 | 重试次数 |
|------|------|------|---------|
| **令牌上限升级** | 1195-1221 | 升级到 64k 令牌上限 | 1次 |
| **恢复消息注入** | 1223-1251 | 注入元消息 `"Output token limit hit. Resume directly..."` | 最多3次 |
| **耗尽退出** | 1254-1256 | 暴露错误并退出 | 0次 |

### 4.6 媒体大小错误（isWithheldMedia）

**文件**: `src/query.ts:1082-1084`

```typescript
const isWithheldMedia =
  mediaRecoveryEnabled &&
  reactiveCompact?.isWithheldMediaSizeError(lastMessage)
```

**处理**：与 413 错误共用 `reactiveCompact.tryReactiveCompact` 路径

**关键特性**：媒体错误**跳过** collapse drain 层，直接进入 reactive compact（因为 collapse 不剥离图片）

### 4.7 流内错误 withhold 机制

**文件**: `src/query.ts:799-822`

以下错误消息在流式循环内部被**扣留**（withhold），不立即 yield 给调用方：

```typescript
let withheld = false
if (contextCollapse?.isWithheldPromptTooLong(message, ...)) withheld = true
if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
if (isWithheldMaxOutputTokens(message)) withheld = true
if (!withheld) { yield yieldMessage }
```

**目的**：防止中间错误泄露给 SDK 调用方，让恢复循环在幕后完成

---

## 5. 工具执行错误处理

### 5.1 错误类定义

**文件**: `src/utils/errors.ts`

| 类名 | 行号 | 用途 | 重试 |
|------|------|------|------|
| `ClaudeError` | 3-8 | 通用基础错误 | 否 |
| `MalformedCommandError` | 10 | 命令格式错误 | 否 |
| `AbortError` | 12-17 | 用户中断 | 否 |
| `ConfigParseError` | 39-49 | 配置文件解析错误 | 否 |
| `ShellError` | 51-61 | Shell 命令失败 | 否 |
| `TeleportOperationError` | 63-71 | 远程操作错误 | 否 |
| `TelemetrySafeError` | 93-101 | 安全可记录到遥测的错误 | 否 |

### 5.2 错误格式化 (`formatError`)

**文件**: `src/utils/toolErrors.ts:5-22`

| 错误类型 | 格式化结果 |
|---------|-----------|
| `AbortError` | `INTERRUPT_MESSAGE_FOR_TOOL_USE` |
| `ShellError` | 退出码 + stderr + stdout |
| 其他 | `error.message`（截断至 10k 字符） |
| 非 Error 值 | `String(error)` |

### 5.3 runToolUse 错误处理主流程

**文件**: `src/services/tools/toolExecution.ts`

#### 工具不存在（行 369-411）

```typescript
// 返回错误消息
content: `<tool_use_error>No such tool available: ${toolName}</tool_use_error>`
is_error: true
```

#### AbortController 中止（行 415-453）

```typescript
content: CANCEL_MESSAGE
is_error: true
// 发送 tengu_tool_use_cancelled 事件
```

#### Zod 输入验证失败（行 614-680）

```typescript
// 使用 formatZodValidationError() 生成人类可读消息
// 区分：缺失参数、意外参数、类型不匹配
```

#### 权限拒绝（行 995-1103）

见本文档第 7 节。

#### 工具执行异常（行 1589-1737）

```typescript
// 1. McpAuthError → 更新 MCP 客户端状态为 needs-auth
if (error instanceof McpAuthError) {
  toolUseContext.setAppState(prevState => {
    const updatedClients = [...prevState.mcp.clients]
    updatedClients[existingClientIndex] = {
      name: serverName,
      type: 'needs-auth' as const,
      config: existingClient.config,
    }
    return { ...prevState, mcp: { ...prevState.mcp, clients: updatedClients } }
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
    content: [{ type: 'tool_result', content, is_error: true, tool_use_id }],
    toolUseResult: `Error: ${content}`,
  }),
  ...hookMessages,
}]
```

### 5.4 错误到遥测的分类 (`classifyToolError`)

**文件**: `src/services/tools/toolExecution.ts:150-171`

| 错误类型 | 遥测标识 |
|---------|---------|
| `TelemetrySafeError` | `telemetryMessage` 字段（最多 200 字符） |
| Node.js fs 错误 | `Error:ENOENT`, `Error:EACCES` 等 |
| 有 `.name` 属性的错误 | `.name`（最多 60 字符） |
| 其他 | `Error` 或 `UnknownError` |

---

## 6. MCP 错误处理

### 6.1 错误类定义

**文件**: `src/services/mcp/client.ts`

| 类名 | 行号 | 用途 |
|------|------|------|
| `McpAuthError` | 152-159 | MCP 认证错误（HTTP 401 / OAuth token 过期） |
| `McpToolCallError` | 177-186 | MCP 工具返回 `isError: true` |
| `McpSessionExpiredError` | 165-170 | 会话过期（404/-32001 或 -32000 连接关闭） |

### 6.2 McpAuthError 处理流程

**抛出位置**: `client.ts:3204-3207`

```typescript
if (errorCode === 401 || e instanceof UnauthorizedError) {
  throw new McpAuthError(serverName, message)
}
```

**处理位置**: `toolExecution.ts:1601-1629`

```typescript
if (error instanceof McpAuthError) {
  toolUseContext.setAppState(prevState => {
    const updatedClients = [...prevState.mcp.clients]
    updatedClients[existingClientIndex] = {
      name: serverName,
      type: 'needs-auth' as const,  // ← UI 显示需要重新授权
      config: existingClient.config,
    }
    return { ...prevState, mcp: { ...prevState.mcp, clients: updatedClients } }
  })
}
```

**重试**：否。此错误不触发重试，而是将服务器状态标记为 `needs-auth`，等待用户手动重新认证。

### 6.3 McpToolCallError 处理流程

**抛出位置**: `client.ts:3144-3148`

**处理位置**: `toolExecution.ts:1729-1732`

```typescript
mcpMeta: toolUseContext.agentId
  ? undefined
  : error instanceof McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
    ? error.mcpMeta
    : undefined,
```

**重试**：否。MCP 工具返回错误即终止执行流程。

### 6.4 McpSessionExpiredError 处理

**抛出位置**: `client.ts:3217-3231`

两种触发条件：
1. 直接 404 + JSON-RPC -32001（服务器返回 "Session not found"）
2. -32000 "Connection closed"（SDK 在 onerror 处理后关闭传输）

```typescript
clearServerCache(name, config)  // 清除连接缓存
throw new McpSessionExpiredError(serverName)
```

**重试**：是。调用者应使用 `ensureConnectedClient` 获取新的客户端实例后重试。

---

## 7. 权限拒绝错误

### 7.1 PermissionDenied 处理

**文件**: `src/services/tools/toolExecution.ts:995-1103`

#### 触发条件

```typescript
if (permissionDecision.behavior !== 'allow') {
  // 记录日志、构建错误消息、执行钩子
}
```

#### 处理流程

1. **记录日志和遥测**（行 996-1022）
   - 记录 `tengu_tool_use_can_use_tool_rejected` 事件
   - 记录工具名称、MCP 服务器类型、URL 等元数据

2. **构建错误消息内容**（行 1023-1027）
   - 如果设置了 `shouldPreventContinuation` 且无详细钩子消息，显示通用消息

3. **创建错误结果消息**（行 1030-1071）
   ```typescript
   content: [{ type: 'text', ... }]
   is_error: true
   ```

4. **执行 PermissionDenied 钩子**（行 1073-1101）
   - 仅当 `TRANSCRIPT_CLASSIFIER` 开启 **且** auto-mode 分类器拒绝时执行
   - 如果任何钩子返回 `retry: true`，添加重试提示消息

### 7.2 权限拒绝可重试条件

```typescript
// 行 1073-1101
if (
  feature('TRANSCRIPT_CLASSIFIER') &&
  decisionReason.type === 'classifier' &&
  decisionReason.classifier === 'auto-mode'
) {
  for await (const hookResult of runPermissionDeniedHooks(...)) {
    if (hookResult.content === 'retry') {
      // 添加重试提示
    }
  }
}
```

**重试**：条件性重试。只有当 `PermissionDenied` hook 返回 `retry: true` 时，模型才被允许重试。

---

## 8. 错误流图

### 8.1 API 错误流

```
API 返回错误
    │
    ▼
┌───────────────────────────────────┐
│  withRetry.ts 拦截                 │
│  shouldRetry() 判断               │
└───────────────────────────────────┘
    │
    ├─ 可重试 ──→ 指数退避 ──→ 重试 API 调用
    │                    │
    │                    └── 失败次数 >= MAX_529_RETRIES ──→ FallbackTriggeredError
    │                                                         │
    │                                                   fallbackModel ──→ query.ts 切换模型整体重新跑
    │                                                   无 fallback ──→ CannotRetryError ──→ 暴露错误
    │
    └─ 不可重试 ──→ CannotRetryError ──→ getAssistantMessageFromError() ──→ createAssistantAPIErrorMessage()
                                                                    │
                                                                    ▼
                                                            yield 给调用方
```

### 8.2 工具执行错误流

```
工具执行
    │
    ▼
runToolUse()
    │
    ├─ 工具不存在 ──→ tool_result(is_error: true) ──→ 返回
    │
    ├─ Abort ──→ CANCEL_MESSAGE ──→ 返回
    │
    ├─ Zod 验证失败 ──→ formatZodValidationError() ──→ 返回
    │
    ├─ 权限拒绝 ──→ PermissionDenied hook ──→ [retry=true?] ─┐
    │                                                        │
    │                                                  Yes ──┴─→ 添加重试提示 ──→ 返回
    │
    ├─ McpAuthError ──→ 更新 MCP needs-auth 状态 ──→ 返回
    │
    └─ 执行异常 ──→ logError() ──→ PostToolUseFailure hooks ──→ formatError() ──→ 返回
```

### 8.3 query.ts 主循环错误流

```
while-true 循环
    │
    ▼
API 调用
    │
    ├─ 成功 ──→ 处理 tool_use 块 ──→ 继续循环
    │
    └─ 失败
        │
        ├─ FallbackTriggeredError ──→ 清空状态 ──→ 切换模型 ──→ continue
        │
        ├─ ImageSizeError/ImageResizeError ──→ 直接返回 {reason: image_error}
        │
        ├─ isWithheld413/媒体错误 ──┐
        │                          ├─ collapse drain → retry
        │                          ├─ reactive compact → retry
        │                          └─ 失败 → 暴露错误
        │
        ├─ max_output_tokens ──┤
        │                      ├─ 升级令牌上限 → retry
        │                      ├─ 注入恢复消息 → retry (最多3次)
        │                      └─ 耗尽 → 暴露错误
        │
        └─ 其他异常 ──→ 补全丢失的 tool_result ──→ createAssistantAPIErrorMessage ──→ 返回
```

---

## 9. 重试策略总结

### 9.1 重试 vs. 直接暴露对照表

| 错误类型 | 重试？ | 策略 | 文件:行号 |
|---------|-------|------|----------|
| 529 Overloaded（< 3次） | **是** | 指数退避，最多3次后触发 fallback | `withRetry.ts:326` |
| 529 + fallback 可用 | **是** | 切换模型整体重新跑 | `query.ts:894-950` |
| 429 限速（非订阅户/企业） | **是** | `Retry-After` 或指数退避 | `withRetry.ts:696` |
| 429 限速（订阅户） | **否** | Fast mode 降级或直接失败 | `withRetry.ts:267` |
| 401/403（CCR 模式） | **是** | JWT 刷新后重试 | `withRetry.ts:696` |
| 401/403（普通） | **是** | 清除缓存后重试 | `withRetry.ts:696` |
| `APIConnectionError` | **是** | 指数退避 | `withRetry.ts:696` |
| 408/409 | **是** | 重试 | `withRetry.ts:696` |
| `max_tokens` 溢出 | **是** | 令牌上限升级/恢复消息注入 | `query.ts:1195-1251` |
| **isWithheld413 (PTL)** | **是** | collapse drain → reactive compact | `query.ts:1085-1166` |
| **媒体大小错误** | **是** | reactive compact | `query.ts:1119-1166` |
| 图片/PDF 错误 | **否** | 提示用户处理 | `query.ts:969` |
| 工具不存在 | **否** | 直接暴露 | `toolExecution.ts:369` |
| 工具执行异常 | **否** | 返回错误消息 | `toolExecution.ts:1589` |
| 权限拒绝 | 条件 | Hook 返回 retry:true 时可重试 | `toolExecution.ts:1073` |
| `McpAuthError` | **否** | 标记 needs-auth，等待用户重新认证 | `toolExecution.ts:1601` |
| `McpToolCallError` | **否** | 直接暴露 | `toolExecution.ts:1589` |
| `McpSessionExpiredError` | **是** | 获取新会话后重试 | `client.ts:3217` |
| 500+ 服务器错误 | **是** | 指数退避重试 | `withRetry.ts:696` |

### 9.2 重试方式分类

#### 整体重新跑（Full Retry）
- **FallbackTriggeredError**：清空所有状态，切换模型，从头开始整个请求
- 适用场景：模型相关错误（529 过载、降级）

#### 调整 Prompt 后重试（Prompt Adjusted Retry）
- **isWithheld413**：通过 collapse drain 或 reactive compact 压缩上下文后继续
- **max_output_tokens**：升级令牌上限或注入恢复消息
- 适用场景：资源限制错误

#### 局部重试（Partial Retry）
- **权限拒绝（retry=true）**：仅重试被拒绝的工具调用
- 适用场景：权限一时性拒绝

#### 不重试（No Retry）
- 大部分工具执行错误、图片/PDF 错误、`McpAuthError`
- 适用场景：用户输入错误或需要人工介入的错误

### 9.3 Persistent 模式特殊行为

当 `CLAUDE_CODE_UNATTENDED_RETRY` 开启时：

| 行为 | 普通模式 | Persistent 模式 |
|------|---------|----------------|
| 429/529 重试次数 | 10 次 | **无限** |
| 最大退避 | 32 秒 | **5 分钟** |
| Reset cap | 无 | **6 小时** |
| 等待期间 | 无反馈 | 每 30 秒 heartbeat 消息 |
| 重试耗尽 | 返回错误 | 继续等待直到 cap |

---

## 附录：错误处理关键文件索引

| 文件 | 关键行号 | 核心功能 |
|------|---------|---------|
| `src/services/api/withRetry.ts` | 326-365 | 529 模型降级处理 |
| `src/services/api/withRetry.ts` | 696-787 | shouldRetry 重试决策 |
| `src/services/api/withRetry.ts` | 530-548 | getRetryDelay 延迟计算 |
| `src/services/api/errors.ts` | 965-1161 | classifyAPIError 错误分类 |
| `src/services/api/errors.ts` | 425-934 | getAssistantMessageFromError 消息生成 |
| `src/query.ts` | 894-950 | FallbackTriggeredError 处理 |
| `src/query.ts` | 955-997 | 一般 catch 块 |
| `src/query.ts` | 1085-1117 | isWithheld413 第一层恢复 |
| `src/query.ts` | 1119-1166 | isWithheld413 第二层恢复 + 媒体错误 |
| `src/query.ts` | 1195-1251 | max_output_tokens 恢复 |
| `src/query.ts` | 799-822 | 流内错误 withhold |
| `src/services/tools/toolExecution.ts` | 369-411 | 工具不存在 |
| `src/services/tools/toolExecution.ts` | 614-680 | Zod 验证失败 |
| `src/services/tools/toolExecution.ts` | 995-1103 | 权限拒绝 |
| `src/services/tools/toolExecution.ts` | 1589-1737 | 工具执行异常 |
| `src/utils/toolErrors.ts` | 5-22 | formatError 格式化 |
| `src/services/mcp/client.ts` | 152-159 | McpAuthError |
| `src/services/mcp/client.ts` | 165-170 | McpSessionExpiredError |
| `src/services/mcp/client.ts` | 177-186 | McpToolCallError |
| `src/services/mcp/client.ts` | 3204-3231 | MCP 错误抛出 |
