# Layer 05: KNOWLEDGE ON DEMAND — 按需知识

## 概述

**KNOWLEDGE ON DEMAND** 是懒加载知识系统，包括 Skills（技能）和 Memory（记忆），在需要时才注入上下文。

Claude Code 维护两个独立的记忆子系统：
1. **Auto Memory (memdir)** - 关于用户/项目的持久化、有类型的记忆
2. **CLAUDE.md (nested_memory)** - 项目/用户指令和规则

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 记忆系统 | `src/memdir/memdir.ts` |
| 记忆类型定义 | `src/memdir/memoryTypes.ts` |
| 记忆路径解析 | `src/memdir/paths.ts` |
| CLAUDE.md 系统 | `src/utils/claudemd.ts` |
| 附件系统 | `src/utils/attachments.ts` |
| 技能工具 | `src/tools/SkillTool/SkillTool.ts` |
| 命令定义 | `src/commands.js` |
| 提示词集成 | `src/constants/prompts.ts` |

---

## Auto Memory 系统 (memdir)

### 记忆存储结构

```
~/.claude/
└── projects/
    └── <sanitized-git-root>/  ← 项目级别
        └── memory/
            ├── MEMORY.md              # 索引文件（入口）
            ├── user_role.md           # 用户记忆
            ├── feedback_preferences.md # 反馈记忆
            ├── project_context.md    # 项目记忆
            └── reference_links.md     # 引用记忆

~/.claude/                          # 不在 git 仓库时的备用
└── memory/
    ├── MEMORY.md
    ├── user_role.md
    └── ...

# 团队记忆 (feature-gated, TEAMMEM)
~/.claude/projects/<slug>/memory/
└── team/
    ├── MEMORY.md              # 团队索引
    ├── coding-standards.md
    └── project-decisions.md
```

**源码**: `src/memdir/paths.ts:223-235` (`getAutoMemPath()`)

### 记忆类型（四类分类法）

```typescript
// src/memdir/memoryTypes.ts:14-21
export const MEMORY_TYPES = [
  'user',
  'feedback',
  'project',
  'reference',
] as const

export type MemoryType = (typeof MEMORY_TYPES)[number]
```

| 类型 | 描述 | 范围 |
|------|------|------|
| **user** | 用户的角色、目标、知识、偏好 | 始终私有 |
| **feedback** | 用户给出的关于避免或重复的指导 | 私有或团队 |
| **project** | 进行中的工作、计划、bug、事件 | 私有或团队 |
| **reference** | 指向外部系统的指针 | 通常团队 |

### 记忆前置格式

```typescript
// src/memdir/memoryTypes.ts:261-271
// 每个记忆文件使用前置格式
---
name: 记忆名称
description: 一行描述，用于在未来对话中判断相关性
type: user|feedback|project|reference
---

记忆内容正文。feedback/project 类型请结构化为：
规则/事实，然后 **Why:** 和 **How to apply:** 行。
```

### loadMemoryPrompt() - 核心加载函数

```typescript
// src/memdir/memdir.ts:419-507
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()

  // KAIROS daily-log 模式优先于 TEAMMEM
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  if (feature('TEAMMEM')) {
    if (teamMemPaths!.isTeamMemoryEnabled()) {
      // 返回组合提示词（私有 + 团队）
      return teamMemPrompts!.buildCombinedMemoryPrompt(extraGuidelines, skipIndex)
    }
  }

  if (autoEnabled) {
    const autoDir = getAutoMemPath()
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines('auto memory', autoDir, extraGuidelines, skipIndex).join('\n')
  }

  return null  // 禁用时返回 null
}
```

### buildMemoryLines() - 构建记忆提示词

```typescript
// src/memdir/memdir.ts:199-266
export function buildMemoryLines(
  displayName: string,
  memoryDir: string,
  extraGuidelines?: string[],
  skipIndex = false,
): string[] {
  const lines: string[] = [
    `# ${displayName}`,
    '',
    `You have a persistent, file-based memory system at \`${memoryDir}\`.`,
    '',
    '## Types of memory',
    '...',  // 四类记忆的详细说明
    '',
    '## How to save memories',
    // 两步流程：
    // 1. 将记忆写入自己的文件，带前置格式
    // 2. 在 MEMORY.md 中添加指针（索引）
    '',
    '## Memory and other forms of persistence',
    // Memory vs Plan vs Task 的使用指南
  ]

  return lines
}
```

---

## CLAUDE.md 系统 (nested_memory)

### 文件发现顺序

```
1. Managed memory        → /etc/claude-code/CLAUDE.md (全局，始终加载)
2. User memory         → ~/.claude/CLAUDE.md (私有全局)
3. Project memory      → ./CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
4. Local memory       → ./CLAUDE.local.md (私有项目级)
5. Memdir 入口  → ~/.claude/projects/.../memory/MEMORY.md
```

**源码**: `src/utils/claudemd.ts:1-26`

### getMemoryFiles() 函数

```typescript
// src/utils/claudemd.ts:790-1000
export const getMemoryFiles = memoize(
  async (forceIncludeExternal: boolean = false): Promise<MemoryFileInfo[]> => {
    // 按顺序处理: Managed → User → Project → Local → AutoMem
    // ...
  }
)
```

此函数：
- 从当前目录向上遍历到根目录
- 发现所有相关的 CLAUDE.md 文件
- 处理记忆文件中的 `@include` 指令
- 返回带内容和元数据的 `MemoryFileInfo[]`

### @include 指令支持

```typescript
// src/utils/claudemd.ts:18-25
// 语法: @path, @./relative/path, @~/home/path, @/absolute/path
// 示例:
// # 在 CLAUDE.md 中
// @./SPEC.md
// @~/notes/architecture.md
```

### nested_memory 附件机制

```typescript
// src/utils/attachments.ts:1710-1775
export function memoryFilesToAttachments(
  memoryFiles: MemoryFileInfo[],
  toolUseContext: ToolUseContext,
  triggerFilePath?: string,
): Attachment[] {
  // 返回 type: 'nested_memory' 的附件
  return attachments
}

// 附件结构：
{
  type: 'nested_memory',
  path: string,
  content: MemoryFileInfo,
  displayPath: string
}
```

### nested_memory 渲染

```typescript
// src/utils/messages.ts:3700-3707
case 'nested_memory': {
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `Contents of ${attachment.content.path}:\n\n${attachment.content.content}`,
      isMeta: true,
    }),
  ])
}
```

---

## 记忆加载流程

### 每轮: 系统提示词记忆章节

```
用户输入
    ↓
QueryEngine.submitMessage()
    ↓
getSystemPrompt() → loadMemoryPrompt()
    ↓
系统提示词包含:
# Memory
You have a persistent, file-based memory system at...
## Types of memory
...
## Memory.md (仅索引内容)
```

**每次 API 调用都发生** — 但只有索引（MEMORY.md），不是完整文件内容。

### 按需: nested_memory 附件

```
模型调用工具（如 Read, Edit）
    ↓
getAttachmentMessages() → getNestedMemoryAttachmentsForFile()
    ↓
检查 loadedNestedMemoryPaths（去重）
    ↓
memoryFilesToAttachments()
    ↓
附件添加到 API 请求
    ↓
模型看到记忆内容
```

### 去重机制

```typescript
// src/utils/attachments.ts:1718-1724
if (toolUseContext.loadedNestedMemoryPaths?.has(memoryFile.path)) {
  continue  // 已经加载，跳过
}
toolUseContext.loadedNestedMemoryPaths?.add(memoryFile.path)
```

---

## Skills 系统

### SkillTool

```typescript
// src/tools/SkillTool/SkillTool.ts
export const SkillTool = buildTool({
  name: SKILL_TOOL_NAME,
  shouldDefer: true,  // 延迟加载

  async call({ skill, arguments_ }, context) {
    // 1. 查找技能命令
    const command = findCommand(skill, allCommands)

    // 2. 获取技能内容
    const prompt = await command.getPromptForCommand(arguments_, context)

    // 3. 执行技能（分叉执行）
    const result = await prepareForkedCommandContext(command, arguments_, context)

    // 4. 创建分叉代理执行
    for await (const message of runAgent({ ... })) {
      yield message
    }
  },
})
```

### 技能定义来源

| 来源 | 说明 |
|------|------|
| **内置技能** | `src/commands.js` 定义的 slash commands |
| ** MCP 技能** | MCP 服务器提供的 prompt 类型命令 |
| **插件技能** | 插件提供的技能 |

### 技能执行流程

```typescript
// src/utils/forkedAgent.ts:191-200
export async function prepareForkedCommandContext(
  command: PromptCommand,
  args: string,
  context: ToolUseContext,
): Promise<PreparedForkedContext> {
  // 1. 获取技能内容（替换 $ARGUMENTS）
  const skillPrompt = await command.getPromptForCommand(args, context)

  // 2. 构建分叉消息
  const skillContent = skillPrompt
    .map(block => (block.type === 'text' ? block.text : ''))
    .join('\n')

  // 3. 返回上下文供 runAgent 使用
  return { skillContent, ... }
}
```

---

## 提示词集成

### 系统提示词中的记忆章节

```typescript
// src/constants/prompts.ts:491-495
const dynamicSections = [
  systemPromptSection('memory', () => loadMemoryPrompt()),
  // ...
]
```

### startRelevantMemoryPrefetch

```typescript
// src/utils/attachments.ts
export function startRelevantMemoryPrefetch(
  messages: Message[],
  toolUseContext: ToolUseContext,
): MemoryPrefetch {
  // 在 API 调用前预取相关记忆
  // 异步执行，不阻塞主流程
}
```

---

## Feature Flags

| Flag | 用途 |
|------|------|
| `TEAMMEM` | 启用团队记忆（团队成员间共享） |
| `KAIROS` | 启用每日日志模式（追加写入日志） |
| `EXTRACT_MEMORIES` | 后台代理提取记忆 |
| `tengu_coral_fern` | 在记忆中启用过去上下文搜索 |
| `tengu_moth_copse` | 跳过 MEMORY.md 索引（仅使用主题文件） |

---

## 记忆文件大小限制

| 限制 | 值 | 用途 |
|------|-----|------|
| `MAX_ENTRYPOINT_LINES` | 200 | MEMORY.md 索引最大行数 |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | MEMORY.md 最大字节数 |
| `MAX_MEMORY_CHARACTER_COUNT` | 40,000 | 记忆文件建议最大大小 |

**源码**: `src/memdir/memdir.ts:34-38`

---

## 设计原则

### 1. 为什么两个记忆系统？

| 系统 | 特点 | 用途 |
|------|------|------|
| **Auto Memory (memdir)** | 类型化、可搜索、跨会话持久化 | 跨对话学习 |
| **CLAUDE.md** | 直接项目指令、可版本控制 | 项目特定规则 |

### 2. 为什么用附件而非系统提示词？

- 系统提示词有大小限制
- 附件可根据上下文条件加载
- 不同文件可在不同时间加载
- LRU + 去重防止无限增长

### 3. 记忆漂移处理

```typescript
// src/memdir/memoryTypes.ts:240-256
// 系统指示在推荐前验证：
// - 如果命名文件路径：检查文件是否存在
// - 如果命名函数或标志：搜索它
// - 如果用户要依据它采取行动：先验证

// "记忆说 X 存在" ≠ "X 存在"
```

---

## 流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                     每次 API 调用                                     │
│                                                                     │
│  系统提示词：                                                        │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ # Memory                                                     │  │
│  │ - 记忆系统说明                                               │  │
│  │ - 四类记忆描述                                               │  │
│  │ - 如何保存记忆                                               │  │
│  │ - MEMORY.md (仅索引内容)                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  消息历史：                                                         │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ [完整对话历史]                                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  附件（按需）：                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ nested_memory: CLAUDE.md 内容（相关时）                     │  │
│  │ relevant_memories: 主题文件内容                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    工具访问时                                        │
│                                                                     │
│  模型调用 Read(file="src/auth/login.ts")                           │
│       ↓                                                             │
│  getNestedMemoryAttachmentsForFile("src/auth/login.ts")             │
│       ↓                                                             │
│  1. 检查匹配 "src/auth/**" 的条件规则                              │
│  2. 检查父目录中的 CLAUDE.md                                       │
│  3. 检查 .claude/rules/*.md                                         │
│       ↓                                                             │
│  memoryFilesToAttachments() → 添加到请求                           │
│       ↓                                                             │
│  模型看到相关项目指令                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| loadMemoryPrompt | `src/memdir/memdir.ts:419-507` |
| buildMemoryLines | `src/memdir/memdir.ts:199-266` |
| getAutoMemPath | `src/memdir/paths.ts:223-235` |
| MEMORY_TYPES | `src/memdir/memoryTypes.ts:14-21` |
| getMemoryFiles | `src/utils/claudemd.ts:790-1000` |
| memoryFilesToAttachments | `src/utils/attachments.ts:1710-1775` |
| getNestedMemoryAttachmentsForFile | `src/utils/attachments.ts:1792-1850` |
| loadMemoryPrompt 集成 | `src/constants/prompts.ts:491-495` |
| nested_memory 渲染 | `src/utils/messages.ts:3700-3707` |
| SkillTool | `src/tools/SkillTool/SkillTool.ts` |
| prepareForkedCommandContext | `src/utils/forkedAgent.ts:191-200` |

---

*基于 Claude Code v2.1.88 源代码分析*
