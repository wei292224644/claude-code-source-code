# MCP (Model Context Protocol) 模块技术文档

## 1. MCP 协议概述

### 1.1 什么是 Model Context Protocol

Model Context Protocol（MCP）是一种标准化协议，用于在大型语言模型（LLM）与外部工具和数据源之间建立通信。MCP 使得 LLM 能够通过统一的接口调用外部工具、访问资源，实现能力扩展。

在 Claude Code 架构中，MCP 是一种**远程工具协议**，与本地 Tool 有着明确的区分：

| 特性 | 本地 Tool | MCP Tool |
|------|----------|----------|
| 调用方式 | 直接函数调用 | 跨进程/跨网络 JSON-RPC |
| 传输协议 | 进程内 | HTTP/SSE/WebSocket/STDIO |
| 工具发现 | 编译时静态注册 | 运行时动态发现 |
| 服务器管理 | 应用生命周期 | 独立进程/服务 |

### 1.2 支持的 Transport 类型

MCP 协议支持多种传输层实现：

```typescript
// src/services/mcp/types.ts
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
```

| Transport 类型 | 描述 | 典型场景 |
|---------------|------|----------|
| `stdio` | 标准输入/输出 | 本地子进程 |
| `sse` | Server-Sent Events + HTTP | 远程 HTTP 服务 |
| `sse-ide` | IDE 专用 SSE | VSCode 插件 |
| `http` | Streamable HTTP | 现代 MCP 服务 |
| `ws` | WebSocket | 实时双向通信 |
| `sdk` | SDK 内嵌传输 | In-Process 模式 |

### 1.3 协议消息格式

MCP 使用 JSON-RPC 2.0 风格的消息格式：

```typescript
// 请求格式
interface JSONRPCRequest {
  jsonrpc: '2.0'
  id: string | number
  method: string
  params?: {
    name: string
    arguments?: Record<string, unknown>
    _meta?: Record<string, unknown>
  }
}

// 响应格式
interface JSONRPCResponse {
  jsonrpc: '2.0'
  id: string | number
  result?: {
    content: Array<{ type: string; text?: string; resource?: unknown }>
    isError?: boolean
  }
  error?: {
    code: number
    message: string
    data?: unknown
  }
}
```

---

## 2. MCP 类型定义

### 2.1 服务器配置模型

MCP 服务器配置使用 Zod Schema 进行验证：

```typescript
// src/services/mcp/types.ts

// STDIO 服务器配置（本地子进程）
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(), // 兼容旧版本
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)

// SSE 服务器配置
export const McpSSEServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sse'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),  // 动态获取 headers 的脚本
    oauth: McpOAuthConfigSchema().optional(),
  }),
)

// HTTP 服务器配置
export const McpHTTPServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('http'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
    oauth: McpOAuthConfigSchema().optional(),
  }),
)

// WebSocket 服务器配置
export const McpWebSocketServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('ws'),
    url: z.string(),
    headers: z.record(z.string(), z.string()).optional(),
    headersHelper: z.string().optional(),
  }),
)

// SDK 服务器配置（内嵌）
export const McpSdkServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('sdk'),
    name: z.string(),
  }),
)
```

### 2.2 OAuth 配置

```typescript
const McpOAuthConfigSchema = lazySchema(() =>
  z.object({
    clientId: z.string().optional(),
    callbackPort: z.number().int().positive().optional(),
    authServerMetadataUrl: z
      .string()
      .url()
      .startsWith('https://', {
        message: 'authServerMetadataUrl must use https://',
      })
      .optional(),
    xaa: z.boolean().optional(),  // Cross-App Access 配置
  }),
)
```

### 2.3 服务器能力（ServerCapabilities）

```typescript
export type ConnectedMCPServer = {
  client: Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities  // MCP SDK 定义的能力对象
  serverInfo?: {
    name: string
    version: string
  }
  instructions?: string  // 服务器提供的使用说明
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>  // 清理函数
}
```

ServerCapabilities 包含的能力：

```typescript
interface ServerCapabilities {
  tools?: {
    listChanged?: boolean
  }
  resources?: {
    subscribe?: boolean
    listChanged?: boolean
  }
  prompts?: {
    listChanged?: boolean
  }
  elicitation?: {
    form?: Record<string, unknown>
    url?: Record<string, unknown>
  }
}
```

### 2.4 服务器连接状态

```typescript
export type MCPServerConnection =
  | ConnectedMCPServer      // 已连接
  | FailedMCPServer        // 连接失败
  | NeedsAuthMCPServer     // 需要认证
  | PendingMCPServer       // 连接中
  | DisabledMCPServer      // 已禁用
```

---

## 3. MCPClient 实现

### 3.1 核心连接流程

`connectToServer` 函数是建立 MCP 连接的核心：

```typescript
// src/services/mcp/client.ts (约第 595-1172 行)

export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
  ): Promise<MCPServerConnection> => {
    const connectStartTime = Date.now()
    let transport: Transport

    // 根据服务器类型创建对应的 Transport
    if (serverRef.type === 'sse') {
      // SSE Transport 实现
      const authProvider = new ClaudeAuthProvider(name, serverRef)
      const combinedHeaders = await getMcpServerHeaders(name, serverRef)

      transport = new SSEClientTransport(new URL(serverRef.url), {
        authProvider,
        fetch: wrapFetchWithTimeout(/* ... */),
        eventSourceInit: {
          fetch: /* 不带超时 */,
        },
      })
    } else if (serverRef.type === 'http') {
      // HTTP Streamable Transport
      transport = new StreamableHTTPClientTransport(
        new URL(proxyUrl),
        transportOptions,
      )
    } else if (serverRef.type === 'ws') {
      // WebSocket Transport
      transport = new WebSocketTransport(wsClient)
    } else if (serverRef.type === 'stdio') {
      // STDIO Transport（子进程）
      transport = new StdioClientTransport({
        command: serverRef.command,
        args: serverRef.args,
        env: { ...subprocessEnv(), ...serverRef.env },
        stderr: 'pipe',
      })
    }

    // 创建 MCP Client 实例
    const client = new Client(
      {
        name: 'claude-code',
        version: MACRO.VERSION,
      },
      {
        capabilities: {
          roots: {},
          elicitation: {},
        },
      },
    )

    // 设置请求处理器
    client.setRequestHandler(ListRootsRequestSchema, async () => ({
      roots: [{ uri: `file://${getOriginalCwd()}` }],
    }))

    // 连接并设置超时
    await Promise.race([
      client.connect(transport),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Connection timeout')), 30000)
      ),
    ])

    // 获取服务器能力
    const capabilities = client.getServerCapabilities()
    const serverVersion = client.getServerVersion()
    const instructions = client.getInstructions()

    return {
      type: 'connected',
      client,
      name,
      capabilities,
      serverInfo: { name: serverVersion.name, version: serverVersion.version },
      instructions,
      config: serverRef,
      cleanup: async () => {
        await client.close()
        await transport.close()
      },
    }
  },
)
```

### 3.2 工具调用流程

```typescript
// src/services/mcp/client.ts (约第 3029-3150 行)

async function callMCPTool({
  client: { client, name, config },
  tool,
  args,
  meta,
  signal,
  onProgress,
}: {
  client: ConnectedMCPServer
  tool: string
  args: Record<string, unknown>
  meta?: Record<string, unknown>
  signal: AbortSignal
  onProgress?: (data: MCPProgress) => void
}): Promise<{
  content: MCPToolResult
  _meta?: Record<string, unknown>
}> {
  const timeoutMs = getMcpToolTimeoutMs()  // 默认 ~27.8 小时

  const result = await Promise.race([
    client.callTool(
      {
        name: tool,
        arguments: args,
        _meta: meta,
      },
      CallToolResultSchema,
      {
        signal,
        timeout: timeoutMs,
        onprogress: onProgress
          ? sdkProgress => onProgress({ /* 转换进度 */ })
          : undefined,
      },
    ),
    // 自定义超时处理
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error('Tool timeout')), timeoutMs)
    ),
  ])

  if ('isError' in result && result.isError) {
    throw new McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
      errorMessage,
      'MCP tool call failed',
      result._meta,
    )
  }

  return {
    content: result.content,
    _meta: result._meta,
  }
}
```

### 3.3 工具列表获取

```typescript
// 工具列表通过 MCP SDK 的 client.listTools() 获取
// 在连接成功后，调用 client.listTools() 获取可用工具

const tools = await client.listTools()
const toolList = tools.map(tool => ({
  name: buildMcpToolName(serverName, tool.name),  // mcp__serverName__toolName
  description: tool.description,
  inputSchema: tool.inputSchema,
  originalToolName: tool.name,  // 保存原始名称用于调用
}))
```

### 3.4 资源订阅（SSE）

```typescript
// 对于支持 subscribe 能力的服务器
if (capabilities.resources?.subscribe) {
  // 通过 SSE 传输订阅资源更新
  client.setRequestHandler(
    SubscribeRequestSchema,
    async ({ uri }) => {
      // 资源订阅处理
    }
  )
}
```

---

## 4. 工具名称归一化

### 4.1 为什么需要归一化

不同 MCP Server 对工具命名有不同的风格：

```
Server A: snake_case → get_user_info
Server B: camelCase  → getUserInfo
Server C: kebab-case → get-user-info
```

Claude Code 内部统一使用 `snake_case`，但 MCP 服务器可能使用任意风格。归一化确保：
1. 工具名称在 Agent 系统中格式统一
2. 支持任意命名风格的 MCP 服务器
3. 避免与 Agent 内部名称冲突

### 4.2 归一化函数实现

```typescript
// src/services/mcp/normalization.ts

// Claude.ai 服务器前缀
const CLAUDEAI_SERVER_PREFIX = 'claude.ai '

/**
 * 将服务器名称转换为符合 API 规范的格式
 * API 规范: ^[a-zA-Z0-9_-]{1,64}$
 * 将无效字符替换为下划线
 * 对于 claude.ai 服务器，还会合并连续下划线并去除首尾下划线
 */
export function normalizeNameForMCP(name: string): string {
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}
```

### 4.3 工具名称构建

```typescript
// src/services/mcp/mcpStringUtils.ts

/**
 * 从工具名称字符串解析 MCP 服务器信息
 * 期望格式: "mcp__serverName__toolName"
 */
export function mcpInfoFromString(toolString: string): {
  serverName: string
  toolName: string | undefined
} | null {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  if (mcpPart !== 'mcp' || !serverName) {
    return null
  }
  const toolName =
    toolNameParts.length > 0 ? toolNameParts.join('__') : undefined
  return { serverName, toolName }
}

/**
 * 生成 MCP 工具的完整名称
 * 格式: mcp__${normalizeName(serverName)}__${normalizeName(toolName)}
 */
export function buildMcpToolName(serverName: string, toolName: string): string {
  return `${getMcpPrefix(serverName)}${normalizeNameForMCP(toolName)}`
}

export function getMcpPrefix(serverName: string): string {
  return `mcp__${normalizeNameForMCP(serverName)}__`
}
```

### 4.4 参数 Schema 归一化

MCP 服务器返回的工具参数 schema 可能包含非标准格式，归一化处理：

```typescript
// 典型的 MCP 工具 inputSchema
{
  "type": "object",
  "properties": {
    "userId": { "type": "string", "description": "User identifier" },
    "page_size": { "type": "integer", "description": "Items per page" }
  },
  "required": ["userId"]
}

// 归一化处理后转换为 Agent 内部格式
```

---

## 5. MCP 工具到 Agent Tool 的桥接

### 5.1 MCPTool 桥接架构

```typescript
// src/tools/MCPTool/MCPTool.ts

export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 会被 mcpClient.ts 覆盖为完整名称
  maxResultSizeChars: 100_000,

  async call(toolUse: MCPToolUse): Promise<MCPToolResult> {
    // 实际实现在 mcpClient.ts 中
    const { connectedClient, mcpResult } = await callMCPToolWithUrlElicitationRetry({
      client: /* 从 state 获取的 MCPClient */,
      tool: toolUse.name,
      args: toolUse.arguments,
      // ...
    })
    return mcpResult
  },
})
```

### 5.2 工具查找与路由

```typescript
// 工具名称格式: mcp__serverName__toolName
// 例如: mcp__github__create_issue

function findMCPServerForTool(toolName: string): ConnectedMCPServer | null {
  const info = mcpInfoFromString(toolName)
  if (!info) return null

  // 从 AppStateStore 获取对应 server 的客户端
  const client = appState.mcpClients.get(info.serverName)
  return client ?? null
}
```

### 5.3 多服务器路由

当存在多个 MCP 服务器时，工具名称确保唯一性：

```typescript
// 服务器 A 和 B 都提供 "user_info" 工具
// 路由结果:
//   mcp__server_a__user_info  → Server A
//   mcp__server_b__user_info  → Server B
```

---

## 6. MCP Server 配置和管理

### 6.1 配置文件结构

```json
// .mcp.json (项目级)
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 6.2 配置来源优先级

```typescript
// src/services/mcp/config.ts

// 优先级从低到高: plugin < user < project < local
const configs = Object.assign(
  {},
  dedupedPluginServers,   // 最低优先级
  userServers,
  approvedProjectServers,
  localServers,           // 最高优先级
)
```

### 6.3 多实例管理

```typescript
// src/services/mcp/client.ts

// 使用 memoize 缓存连接，实现连接复用
export const connectToServer = memoize(
  async (name: string, serverRef: ScopedMcpServerConfig) => {
    // 连接逻辑...
  },
)

// 获取缓存的连接或创建新连接
export async function ensureConnectedClient(
  client: ConnectedMCPServer,
): Promise<ConnectedMCPServer> {
  // 检查连接是否有效
  if (client.config.type === 'sdk') {
    return client  // SDK 类型始终有效
  }

  // 检查会话是否过期
  if (isMcpSessionExpiredError(error)) {
    // 清除缓存并重连
    connectToServer.clear()
    return connectToServer(client.name, client.config)
  }

  return client
}
```

### 6.4 断线重连

```typescript
// src/services/mcp/client.ts

// 连接超时配置
function getConnectionTimeoutMs(): number {
  return parseInt(process.env.MCP_TIMEOUT || '', 10) || 30000
}

// 工具调用超时（默认 ~27.8 小时）
function getMcpToolTimeoutMs(): number {
  return (
    parseInt(process.env.MCP_TOOL_TIMEOUT || '', 10) ||
    DEFAULT_MCP_TOOL_TIMEOUT_MS  // 100_000_000 ms
  )
}

// 会话过期检测
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? error.code : undefined
  if (httpStatus !== 404) return false
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
```

---

## 7. MCP 协议完整流程

### 7.1 连接建立流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      MCP Client 启动                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 读取 MCP 配置                                                │
│     - .mcp.json (项目级)                                        │
│     - 全局配置 (用户级)                                          │
│     - 企业配置 (最高优先级)                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 创建 Transport                                               │
│     ┌──────────────┬──────────────┬──────────────┐              │
│     │ stdio        │ sse/http     │ ws           │              │
│     │ StdioClient  │ SSEClient    │ WebSocket    │              │
│     │ Transport    │ Transport    │ Transport    │              │
│     └──────────────┴──────────────┴──────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 建立连接 (connect)                                          │
│     - 发送 JSON-RPC initialize 请求                             │
│     - 携带客户端能力 (capabilities)                              │
│     - 获取服务器能力回应                                         │
│     - 设置请求处理器 (ListRoots, ElicitRequest)                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 获取资源列表 (list_tools / list_resources)                  │
│     - client.listTools() → Tool[]                               │
│     - client.listResources() → Resource[]                       │
│     - 归一化工具名称                                             │
│     - 注册到 Tool Registry                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 连接就绪                                                     │
│     - MCPServerConnection.type = 'connected'                    │
│     - 可接受工具调用                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 工具调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      Agent 请求工具调用                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 解析工具名称                                                 │
│     mcp__github__create_issue                                   │
│     → server: github, tool: create_issue                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 查找 MCP Client                                             │
│     - 从 AppState.mcpClients 获取                               │
│     - 若不存在，触发连接                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 发送工具调用请求                                             │
│     client.callTool({                                          │
│       name: 'create_issue',                                     │
│       arguments: { owner, repo, title, body },                   │
│     })                                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 处理响应                                                     │
│     - 成功: 返回 content[]                                       │
│     - 错误: 抛出 McpToolCallError                                │
│     - 认证失败: 触发重新认证流程                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 转换结果格式                                                 │
│     - MCP content[] → Agent ToolResult                          │
│     - 支持进度回调 (onProgress)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 SSE 资源订阅流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   资源订阅初始化                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 检查服务器能力                                               │
│     capabilities.resources?.subscribe === true                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 订阅资源                                                     │
│     client.subscribeResource({ uri: 'file://path/to/resource' })│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. SSE 事件接收                                                 │
│     - EventSource 监听 server-sent events                       │
│     - 事件类型: resource.updated, resource.deleted               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 回调处理                                                     │
│     onResourceUpdated(uri, content) → 通知 Agent                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 安全考量

### 8.1 Header 中的 API Key 传递

```typescript
// src/services/mcp/headersHelper.ts

// 动态获取 headers（用于敏感信息）
export async function getMcpHeadersFromHelper(
  serverName: string,
  config: McpSSEServerConfig | McpHTTPServerConfig,
): Promise<Record<string, string> | null> {
  if (!config.headersHelper) {
    return null
  }

  // 安全检查：项目级配置需要信任确认
  if (isMcpServerFromProjectOrLocalSettings(config) && !isNonInteractive()) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      logEvent('tengu_mcp_headersHelper_missing_trust', {})
      return null  // 阻止在未确认信任前执行
    }
  }

  // 执行 headersHelper 脚本
  const execResult = await execFileNoThrowWithCwd(config.headersHelper, [], {
    shell: true,
    timeout: 10000,
    env: {
      ...process.env,
      CLAUDE_CODE_MCP_SERVER_NAME: serverName,
      CLAUDE_CODE_MCP_SERVER_URL: config.url,
    },
  })

  // 验证返回格式
  const headers = jsonParse(execResult.stdout)
  // 确保所有值都是字符串
  return headers as Record<string, string>
}

// 合并静态和动态 headers
export async function getMcpServerHeaders(
  serverName: string,
  config: McpSSEServerConfig | McpHTTPServerConfig,
): Promise<Record<string, string>> {
  const staticHeaders = config.headers || {}
  const dynamicHeaders = await getMcpHeadersFromHelper(serverName, config) || {}

  // 动态 headers 覆盖静态 headers
  return { ...staticHeaders, ...dynamicHeaders }
}
```

### 8.2 URL 验证防止 SSRF

```typescript
// src/services/mcp/config.ts

// CCR 代理 URL 标记检测
const CCR_PROXY_PATH_MARKERS = [
  '/v2/session_ingress/shttp/mcp/',
  '/v2/ccr-sessions/',
]

// 从代理 URL 中提取原始 vendor URL
export function unwrapCcrProxyUrl(url: string): string {
  if (!CCR_PROXY_PATH_MARKERS.some(m => url.includes(m))) {
    return url
  }
  try {
    const parsed = new URL(url)
    const original = parsed.searchParams.get('mcp_url')
    return original || url
  } catch {
    return url
  }
}

// 服务器签名计算（用于去重）
export function getMcpServerSignature(config: McpServerConfig): string | null {
  const cmd = getServerCommandArray(config)
  if (cmd) {
    return `stdio:${jsonStringify(cmd)}`  // 基于命令内容
  }
  const url = getServerUrl(config)
  if (url) {
    return `url:${unwrapCcrProxyUrl(url)}`  // 基于 URL
  }
  return null
}
```

### 8.3 敏感参数脱敏

```typescript
// src/services/mcp/client.ts

// 工具调用时对敏感参数进行脱敏日志
function logMCPDebug(name: string, message: string): void {
  // Authorization header 值被替换为 [REDACTED]
  const wsHeadersForLogging = mapValues(wsHeaders, (value, key) =>
    key.toLowerCase() === 'authorization' ? '[REDACTED]' : value,
  )
}

// OAuth Token 处理
export async function createClaudeAiProxyFetch(
  innerFetch: FetchLike,
): FetchLike {
  return async (url, init) => {
    await checkAndRefreshOAuthTokenIfNeeded()
    const currentTokens = getClaudeAIOAuthTokens()

    const headers = new Headers(init?.headers)
    headers.set('Authorization', `Bearer ${currentTokens.accessToken}`)

    const response = await innerFetch(url, { ...init, headers })

    // 401 处理：刷新 token 后重试
    if (response.status === 401) {
      const tokenChanged = await handleOAuth401Error(sentToken)
      if (tokenChanged) {
        return doRequest()  // 使用新 token 重试
      }
    }

    return response
  }
}
```

### 8.4 企业策略控制

```typescript
// src/services/mcp/config.ts

// 白名单/黑名单策略
export function isMcpServerAllowedByPolicy(
  serverName: string,
  config?: McpServerConfig,
): boolean {
  // 黑名单优先检查
  if (isMcpServerDenied(serverName, config)) {
    return false
  }

  // 无白名单则允许
  const settings = getInitialSettings()
  if (!settings.allowedMcpServers) {
    return true
  }

  // 空的允许列表 = 阻止所有
  if (settings.allowedMcpServers.length === 0) {
    return false
  }

  // 检查命令/URL 模式匹配
  // ...
}
```

---

## 9. 完整 Python 实现示例

虽然 Claude Code 的 MCP 模块使用 TypeScript 实现，以下是一个 Python 等效示例，展示核心逻辑：

```python
"""
MCP Client Python 实现示例
使用 httpx 实现 HTTP/SSE 传输
"""

import asyncio
import json
import uuid
from dataclasses import dataclass
from typing import (
    Any,
    AsyncGenerator,
    Callable,
    Optional,
)
from enum import Enum

import httpx


class TransportType(Enum):
    HTTP = "http"
    SSE = "sse"
    STDIO = "stdio"


@dataclass
class MCPServerConfig:
    name: str
    type: TransportType
    url: Optional[str] = None
    command: Optional[str] = None
    args: list[str] = None
    headers: dict[str, str] = None


@dataclass
class JSONRPCRequest:
    jsonrpc: str = "2.0"
    id: str = None
    method: str = None
    params: dict[str, Any] = None


@dataclass
class JSONRPCResponse:
    jsonrpc: str = "2.0"
    id: str = None
    result: dict[str, Any] = None
    error: dict[str, Any] = None


class MCPClient:
    """
    MCP 客户端实现

    支持 HTTP 和 SSE 传输
    """

    def __init__(self, config: MCPServerConfig):
        self.config = config
        self.client: Optional[httpx.AsyncClient] = None
        self.session_id: Optional[str] = None
        self.capabilities: Optional[dict] = None

    async def connect(self) -> dict:
        """
        建立 MCP 连接

        发送 initialize 请求，获取服务器能力
        """
        self.client = httpx.AsyncClient(
            headers=self.config.headers or {},
            timeout=httpx.Timeout(30.0, connect=10.0),
        )

        if self.config.type == TransportType.HTTP:
            return await self._connect_http()
        elif self.config.type == TransportType.SSE:
            return await self._connect_sse()
        else:
            raise ValueError(f"Unsupported transport type: {self.config.type}")

    async def _connect_http(self) -> dict:
        """HTTP Streamable 连接"""
        # 发送 initialize 请求
        request = JSONRPCRequest(
            id=str(uuid.uuid4()),
            method="initialize",
            params={
                "protocolVersion": "2025-03-26",
                "clientInfo": {
                    "name": "claude-code",
                    "version": "2.1.88",
                },
                "capabilities": {
                    "roots": {},
                    "elicitation": {},
                },
            },
        )

        response = await self._send_request(request)

        if "error" in response:
            raise Exception(f"MCP initialize failed: {response['error']}")

        self.capabilities = response.get("result", {}).get("capabilities", {})
        self.session_id = response.get("result", {}).get("sessionId")

        return self.capabilities

    async def _connect_sse(self) -> dict:
        """SSE 连接 - 用于服务端推送"""
        # SSE 使用不同的连接方式
        async with self.client.stream(
            "GET",
            self.config.url,
            headers={
                **self.config.headers,
                "Accept": "text/event-stream",
            },
        ) as response:
            # 读取 SSE 流直到收到 initialize 结果
            async for line in response.aiter_lines():
                if line.startswith("data:"):
                    data = json.loads(line[5:])
                    if "method" in data and data.get("method") == "initialize":
                        self.capabilities = data.get("params", {}).get(
                            "capabilities", {}
                        )
                        return self.capabilities

    async def _send_request(self, request: JSONRPCRequest) -> dict:
        """发送 JSON-RPC 请求"""
        if self.config.type == TransportType.HTTP:
            # HTTP POST 请求
            payload = {
                "jsonrpc": request.jsonrpc,
                "id": request.id,
                "method": request.method,
                "params": request.params,
            }

            # 如果服务器需要 session，添加到 URL
            url = self.config.url
            if self.session_id:
                url = f"{url}?sessionId={self.session_id}"

            response = await self.client.post(
                url,
                json=payload,
                headers={
                    "Content-Type": "application/json",
                    "Accept": "application/json, text/event-stream",
                },
            )
            response.raise_for_status()
            return response.json()

        else:
            raise NotImplementedError("SSE request sending not implemented")

    async def list_tools(self) -> list[dict]:
        """列出所有可用工具"""
        request = JSONRPCRequest(
            id=str(uuid.uuid4()),
            method="tools/list",
        )

        response = await self._send_request(request)
        return response.get("result", {}).get("tools", [])

    async def call_tool(
        self,
        name: str,
        arguments: dict[str, Any],
        timeout: float = 100000.0,
    ) -> dict:
        """调用 MCP 工具"""
        request = JSONRPCRequest(
            id=str(uuid.uuid4()),
            method="tools/call",
            params={
                "name": name,
                "arguments": arguments,
            },
        )

        response = await self._send_request(request)

        if "error" in response:
            error = response["error"]
            raise Exception(
                f"Tool call failed: {error.get('message', 'Unknown error')}"
            )

        return response.get("result", {})

    async def list_resources(self) -> list[dict]:
        """列出所有可用资源"""
        request = JSONRPCRequest(
            id=str(uuid.uuid4()),
            method="resources/list",
        )

        response = await self._send_request(request)
        return response.get("result", {}).get("resources", [])

    async def subscribe_resource(
        self,
        uri: str,
        callback: Callable[[dict], None],
    ) -> None:
        """订阅资源更新（SSE）"""
        request = JSONRPCRequest(
            id=str(uuid.uuid4()),
            method="resources/subscribe",
            params={"uri": uri},
        )

        # SSE 流式接收更新
        async with self.client.stream(
            "GET",
            self.config.url,
            headers={
                **self.config.headers,
                "Accept": "text/event-stream",
            },
        ) as response:
            async for line in response.aiter_lines():
                if line.startswith("data:"):
                    data = json.loads(line[5:])
                    if data.get("method") == "notifications/resources/updated":
                        resource_data = data.get("params", {})
                        callback(resource_data)

    async def disconnect(self) -> None:
        """优雅关闭连接"""
        if self.client:
            await self.client.aclose()
            self.client = None
            self.session_id = None


# 使用示例
async def main():
    config = MCPServerConfig(
        name="example-server",
        type=TransportType.HTTP,
        url="https://api.example.com/mcp",
        headers={"Authorization": "Bearer ${API_KEY}"},
    )

    client = MCPClient(config)

    # 连接
    capabilities = await client.connect()
    print(f"Server capabilities: {capabilities}")

    # 列出工具
    tools = await client.list_tools()
    print(f"Available tools: {[t['name'] for t in tools]}")

    # 调用工具
    result = await client.call_tool(
        name="get_weather",
        arguments={"city": "Beijing"},
    )
    print(f"Tool result: {result}")

    # 断开连接
    await client.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 10. 目录结构

```
src/services/mcp/
├── auth.ts                    # OAuth 认证处理
├── channelAllowlist.ts       # 通道白名单
├── channelNotification.ts     # 通道通知
├── channelPermissions.ts     # 通道权限
├── claudeai.ts               # Claude.ai MCP 集成
├── config.ts                 # 服务器配置管理
├── elicitationHandler.ts     # elicitation 请求处理
├── envExpansion.ts           # 环境变量展开
├── headersHelper.ts          # 动态 headers 获取
├── InProcessTransport.ts     # 进程内传输（测试用）
├── mcpStringUtils.ts         # 字符串工具函数
├── MCPConnectionManager.tsx  # 连接管理器（React）
├── normalization.ts          # 名称归一化
├── oauthPort.ts              # OAuth 端口
├── officialRegistry.ts       # 官方服务器注册表
├── SdkControlTransport.ts    # SDK 控制传输
├── types.ts                  # 类型定义
├── useManageMCPConnections.ts # React Hook
├── utils.ts                  # 工具函数
├── vscodeSdkMcp.ts          # VSCode SDK MCP
├── xaa.ts                    # Cross-App Access
└── xaaIdpLogin.ts           # XAA IdP 登录
```

---

## 12. 连接生命周期管理

> **重要：修复了原始设计中缺少运行时连接生命周期管理的问题。**

### 12.1 完整状态机

```
                    ┌──────────────┐
                    │   Disabled   │ (配置中已禁用)
                    └──────┬───────┘
                           │ enable()
                           ▼
              ┌────────────────────────┐
              │    Connecting          │◄── retry after failure
              │  (正在建立连接)         │    exponential backoff
              └────────┬───────────────┘
                       │ connected OK
                       ▼
              ┌────────────────────────┐
              │    Connected           │ ◀ 正常运行态
              │  (连接就绪，可调用工具)   │
              └────────┬───────────────┘
                       │ disconnect / error
                       ▼
              ┌────────────────────────┐
              │   Disconnecting        │
              │  (优雅关闭中)           │
              └────────┬───────────────┘
                       │ closed
                       ▼
              ┌────────────────────────┐
              │    Disconnected        │
              │  (连接已关闭)           │
              └────────────────────────┘
```

### 12.2 连接重建策略

SSE 和 HTTP transport 的网络中断需要自动重连：

```python
import asyncio
import random
from dataclasses import dataclass, field


@dataclass
class ReconnectConfig:
    max_retries: int = 5
    initial_delay_ms: int = 1000
    max_delay_ms: int = 60000
    jitter: bool = True


class MCPConnectionManager:
    """
    MCP 连接管理器。
    
    职责：
    1. 连接建立、维持、断开
    2. 断线自动重连（含指数退避）
    3. 连接健康检查（定期 ping）
    4. 并发工具调用的排队与拒绝策略
    """

    def __init__(self, config: MCPServerConfig, reconnect_config: ReconnectConfig | None = None):
        self.config = config
        self.reconnect_config = reconnect_config or ReconnectConfig()
        self._client: Optional[MCPClient] = None
        self._status: str = "disconnected"
        self._reconnect_task: asyncio.Task | None = None
        self._health_check_task: asyncio.Task | None = None
        self._pending_tool_calls: list[tuple[str, dict]] = []  # 连接断开时的等待队列
        self._lock = asyncio.Lock()

    async def connect(self) -> dict:
        """建立连接"""
        async with self._lock:
            self._status = "connecting"
            try:
                self._client = MCPClient(self.config)
                capabilities = await asyncio.wait_for(
                    self._client.connect(),
                    timeout=30.0,
                )
                self._status = "connected"
                # 启动后台维护任务
                self._health_check_task = asyncio.create_task(self._periodic_health_check())
                return capabilities
            except Exception as e:
                self._status = f"failed: {e}"
                raise

    async def _periodic_health_check(self) -> None:
        """
        每 60 秒检查一次连接是否存活。
        
        对于 stdio transport，通过发送一个空 tools/call 来检测子进程是否还活着。
        对于 HTTP/SSE，通过简单的 RPC ping 检测。
        """
        interval = 60  # seconds
        while self._status == "connected":
            await asyncio.sleep(interval)
            try:
                if self._client:
                    await asyncio.wait_for(
                        self._client.call_tool("_health_ping", {}),
                        timeout=5.0,
                    )
            except (asyncio.TimeoutError, Exception):
                # 连接可能已死，触发重连
                self._status = "connecting"
                asyncio.create_task(self._reconnect_with_backoff())

    async def _reconnect_with_backoff(self) -> None:
        """指数退避重连"""
        config = self.reconnect_config
        for attempt in range(config.max_retries):
            delay = min(
                config.initial_delay_ms * (2.0 ** attempt),
                config.max_delay_ms,
            )
            if config.jitter:
                delay = delay * (0.5 + random.random())

            self._status = f"reconnecting (attempt {attempt + 1}/{config.max_retries})"
            await asyncio.sleep(delay / 1000)

            try:
                if self._client:
                    await self._client.disconnect()
            except Exception:
                pass  # 忽略断连失败

            try:
                self._client = MCPClient(self.config)
                await self._client.connect()
                self._status = "connected"
                self._health_check_task = asyncio.create_task(
                    self._periodic_health_check()
                )
                # 重连成功后处理等待队列
                await self._drain_pending_queue()
                return
            except Exception:
                continue

        self._status = "permanently_failed"

    async def call_tool_safe(
        self, tool_name: str, args: dict
    ) -> dict:
        """
        安全调用工具，连接断开时进入等待队列而非立即失败。
        """
        async with self._lock:
            if self._status == "connected" and self._client:
                return await self._client.call_tool(tool_name, args)

            if self._status.startswith("reconnecting"):
                # 加入等待队列，由重连完成后的 drain 处理
                future = asyncio.get_event_loop().create_future()
                self._pending_tool_calls.append((tool_name, args))
                # TODO: 将 future 挂到 drain 逻辑上
                raise MCPConnectionUnavailable(
                    f"Reconnecting, will retry after recovery"
                )

            raise MCPConnectionUnavailable(
                f"Connection is {self._status}"
            )

    async def _drain_pending_queue(self) -> None:
        """重连后处理等待中的工具调用"""
        calls = self._pending_tool_calls
        self._pending_tool_calls = []
        for tool_name, args in calls:
            try:
                if self._client:
                    await self._client.call_tool(tool_name, args)
            except Exception as e:
                # 记录失败但不抛出，避免影响其他调用
                print(f"[MCP] Failed pending call {tool_name}: {e}")

    async def disconnect(self) -> None:
        """优雅关闭"""
        self._status = "disconnecting"
        if self._health_check_task:
            self._health_check_task.cancel()
        if self._client:
            await self._client.disconnect()
        self._status = "disconnected"
```

### 12.3 OAuth Token 刷新流程

```python
async def handle_oauth_401_error(client: MCPClient) -> bool:
    """
    当收到 401 响应时触发 token 刷新。
    
    流程：
    1. 捕获 HTTP 401 响应
    2. 使用 refresh_token 调用 OAuth provider 获取新 access_token
    3. 更新 client 的 Authorization header
    4. 重试失败的请求
    """
    provider = get_oauth_provider(client.server_name)
    tokens = provider.get_tokens()

    if not tokens.refresh_token:
        # 无 refresh token，需要用户重新认证
        trigger_user_auth_flow(client.server_name)
        return False

    try:
        new_tokens = await provider.refresh_token(tokens.refresh_token)
        provider.store_tokens(new_tokens)
        client.update_auth_header(new_tokens.access_token)
        return True
    except Exception as e:
        # refresh 也失败，清除本地存储的 token
        provider.clear_tokens()
        trigger_user_auth_flow(client.server_name)
        return False
```

### 12.4 stdio 子进程管理

```python
class StdioTransport:
    """
    STDIO Transport 的特殊考量。
    
    子进程可能因为以下原因退出：
    1. 正常退出（不应发生，MCP server 应持续运行）
    2. crash / segfault
    3. 被 kill signal 终止
    4. OOM killer
    """

    async def start(self) -> None:
        self._process = await asyncio.create_subprocess_exec(
            self.command,
            *self.args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        # 监控子进程退出
        self._monitor_task = asyncio.create_task(self._monitor_process())

    async def _monitor_process(self) -> None:
        """后台监控子进程是否还在运行"""
        while True:
            retcode = self._process.returncode
            if retcode is not None:
                # 子进程已退出
                if retcode != 0:
                    stderr_output = await self._process.stderr.read()
                    log_event(
                        "mcp_stdio_process_exited",
                        code=retcode,
                        stderr=stderr_output[:500],
                    )
                # 触发重连逻辑
                await self.connection_manager._reconnect_with_backoff()
                return
            await asyncio.sleep(2)  # 每 2 秒检查一次
```

---

## 13. 环境变量

| 变量名 | 默认值 | 描述 |
|--------|--------|------|
| `MCP_TIMEOUT` | `30000` | 连接超时（毫秒） |
| `MCP_TOOL_TIMEOUT` | `100000000` | 工具调用超时（毫秒） |
| `MCP_SERVER_CONNECTION_BATCH_SIZE` | `3` | 本地服务器批量连接大小 |
| `MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE` | `20` | 远程服务器批量连接大小 |
| `HTTP_PROXY` | - | HTTP 代理地址 |
| `HTTPS_PROXY` | - | HTTPS 代理地址 |
| `NO_PROXY` | - | 跳过代理的地址 |
