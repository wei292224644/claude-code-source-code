# Claude Code Memory System - Technical Analysis

## Overview

Claude Code implements a sophisticated multi-layered memory system that persists information across conversations. This document provides a detailed technical analysis of how memory works in Claude Code v2.1.88.

---

## Memory Architecture

Claude Code maintains **two distinct memory subsystems** that serve different purposes:

| Subsystem | Purpose | Location | Loading Strategy |
|-----------|---------|----------|-----------------|
| **Auto Memory (memdir)** | Persistent, typed memories about users/projects | `~/.claude/projects/<slug>/memory/` | System prompt section (every turn) |
| **CLAUDE.md (nested_memory)** | Project/user instructions and rules | Various locations | Attachment-based, on-demand |

---

## 1. Auto Memory System (memdir)

### 1.1 Memory Storage Structure

```
~/.claude/
└── projects/
    └── <sanitized-git-root>/  ← Project-specific
        └── memory/
            ├── MEMORY.md              # Index file (entrypoint)
            ├── user_role.md           # User memories
            ├── feedback_preferences.md # Feedback memories
            ├── project_context.md    # Project memories
            └── reference_links.md     # Reference memories

~/.claude/                          # Fallback if not in git repo
└── memory/
    ├── MEMORY.md
    ├── user_role.md
    └── ...

# Team memory (feature-gated, TEAMMEM)
~/.claude/projects/<slug>/memory/
└── team/
    ├── MEMORY.md              # Team index
    ├── coding-standards.md
    └── project-decisions.md
```

**Source**: `src/memdir/paths.ts:223-235` (`getAutoMemPath()`)

### 1.2 Memory Types (Four-Type Taxonomy)

The memory system is constrained to four types defined in `src/memdir/memoryTypes.ts`:

```typescript
const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| Type | Description | Scope |
|------|-------------|-------|
| **user** | User's role, goals, knowledge, preferences | Always private |
| **feedback** | Guidance from user about what to avoid or repeat | Private or team |
| **project** | Ongoing work, initiatives, bugs, incidents | Private or team |
| **reference** | Pointers to external systems | Usually team |

**Source**: `src/memdir/memoryTypes.ts:14-21`

### 1.3 Memory Frontmatter Format

Each memory file uses frontmatter:

```markdown
---
name: memory name
description: one-line description for relevance matching
type: user|feedback|project|reference
---

Memory content body. For feedback/project types, structure as:
rule/fact, then **Why:** and **How to apply:** lines.
```

**Source**: `src/memdir/memoryTypes.ts:261-271`

### 1.4 How Memory is Saved

The system prompt instructs the model on a **two-step process**:

1. Write memory to its own file with frontmatter
2. Add a pointer to `MEMORY.md` (the index)

```
MEMORY.md is an INDEX, not a memory — each entry one line:
- [Title](file.md) — one-line hook
```

**Source**: `src/memdir/memdir.ts:199-266` (`buildMemoryLines()`)

### 1.5 Memory Loading: `loadMemoryPrompt()`

The core function `loadMemoryPrompt()` (line 419) returns memory instructions for the system prompt:

```typescript
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()
  // ...
  if (autoEnabled) {
    const autoDir = getAutoMemPath()
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines('auto memory', autoDir, extraGuidelines).join('\n')
  }
  return null
}
```

**Source**: `src/memdir/memdir.ts:419-507`

### 1.6 Memory Prompt Integration

In `src/constants/prompts.ts:495`, memory is loaded as a **dynamic section**:

```typescript
const dynamicSections = [
  systemPromptSection('memory', () => loadMemoryPrompt()),
  // ...
]
```

This means:
- **Every API call includes memory instructions in the system prompt**
- The memory **index** (MEMORY.md content) is included, not full file contents
- Detailed memories are read **on-demand** when relevant

**Source**: `src/constants/prompts.ts:491-495`

### 1.7 Memory Search: Past Context

When `tengu_coral_fern` feature flag is enabled, memory search instructions are included:

```typescript
export function buildSearchingPastContextSection(autoMemDir: string): string[] {
  return [
    '## Searching past context',
    `1. Search topic files: grep -rn "<term>" ${autoMemDir} --include="*.md"`,
    `2. Session transcripts: grep -rn "<term>" <project>/ --include="*.jsonl"`,
  ]
}
```

**Source**: `src/memdir/memdir.ts:375-407`

---

## 2. CLAUDE.md System (nested_memory)

### 2.1 File Discovery Order

Files are loaded in **reverse priority order** (highest priority last):

```
1. Managed memory        → /etc/claude-code/CLAUDE.md (global, always loaded)
2. User memory         → ~/.claude/CLAUDE.md (private global)
3. Project memory      → ./CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md
4. Local memory       → ./CLAUDE.local.md (private project-specific)
5. Memdir entrypoint  → ~/.claude/projects/.../memory/MEMORY.md
```

**Source**: `src/utils/claudemd.ts:1-26`

### 2.2 The `getMemoryFiles()` Function

```typescript
export const getMemoryFiles = memoize(
  async (forceIncludeExternal: boolean = false): Promise<MemoryFileInfo[]> => {
    // Process in order: Managed → User → Project → Local → AutoMem
    // ...
  }
)
```

This function:
- Traverses from current directory up to root
- Discovers all relevant CLAUDE.md files
- Handles `@include` directives within memory files
- Returns `MemoryFileInfo[]` with content, type, and metadata

**Source**: `src/utils/claudemd.ts:790-1000`

### 2.3 Processing Memory Files

Each file is processed with `processMemoryFile()`:

```typescript
async function processMemoryFile(
  filePath: string,
  type: MemoryType,
  processedPaths: Set<string>,
  includeExternal: boolean
): Promise<MemoryFileInfo[]>
```

Features:
- Frontmatter stripping (config removed from content)
- `@include` directive resolution
- Circular reference prevention
- Path validation and security checks

**Source**: `src/utils/claudemd.ts:790-1000`

### 2.4 `@include` Directive Support

Memory files can include other files:

```
Syntax: @path, @./relative/path, @~/home/path, @/absolute/path

Example:
# In CLAUDE.md
@./SPEC.md
@~/notes/architecture.md
```

- Works in leaf text nodes only (not inside code blocks)
- Circular references are prevented via `processedPaths` tracking
- Non-existent files are silently ignored

**Source**: `src/utils/claudemd.ts:18-25`

### 2.5 nested_memory Attachment Mechanism

When memory files are loaded, they're converted to **attachments**:

```typescript
export function memoryFilesToAttachments(
  memoryFiles: MemoryFileInfo[],
  toolUseContext: ToolUseContext,
  triggerFilePath?: string,
): Attachment[] {
  // Returns attachments with type: 'nested_memory'
  return attachments
}
```

The attachment structure:
```typescript
{
  type: 'nested_memory',
  path: string,
  content: MemoryFileInfo,
  displayPath: string
}
```

**Source**: `src/utils/attachments.ts:1710-1775`

### 2.6 nested_memory Rendering

In `src/utils/messages.ts:3700-3707`:

```typescript
case 'nested_memory': {
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `Contents of ${attachment.content.path}:\n\n${attachment.content.content}`,
      isMeta: true,
    }),
  ])
}
```

Memory files are rendered as **system reminders** with `isMeta: true`.

**Source**: `src/utils/messages.ts:3700-3707`

### 2.7 Conditional Rules (Glob Matching)

CLAUDE.md files can contain conditional rules with glob patterns:

```markdown
---
globs: ["src/**/*.ts", "src/**/*.tsx"]
---

This rule applies only to TypeScript files.
```

When the model accesses a file matching the glob, the rule is loaded as a **nested_memory** attachment.

**Source**: `src/utils/claudemd.ts:1249-1280` (`getMemoryFilesForNestedDirectory()`)

---

## 3. Memory Loading Flow

### 3.1 Every Turn: System Prompt Memory Section

```
User Input
    ↓
QueryEngine.submitMessage()
    ↓
getSystemPrompt() → loadMemoryPrompt()
    ↓
System Prompt includes:
# Memory
You have a persistent, file-based memory system at...
## Types of memory
...
## How to save memories
...
## Memory.md
[index content]
```

**This happens EVERY turn** - but only the index (MEMORY.md), not full file contents.

### 3.2 On-Demand: nested_memory Attachments

```
Model calls a tool (e.g., Read, Edit)
    ↓
getAttachmentMessages() → getNestedMemoryAttachmentsForFile()
    ↓
Check loadedNestedMemoryPaths (dedup)
    ↓
memoryFilesToAttachments()
    ↓
Attachments added to API request
    ↓
Model sees memory content
```

**Source**: `src/utils/attachments.ts:1792-1850`

### 3.3 Deduplication: `loadedNestedMemoryPaths`

To prevent re-injecting the same memory file multiple times:

```typescript
if (toolUseContext.loadedNestedMemoryPaths?.has(memoryFile.path)) {
  continue  // Already loaded, skip
}
toolUseContext.loadedNestedMemoryPaths?.add(memoryFile.path)
```

Combined with `readFileState` LRU cache for cross-function dedup.

**Source**: `src/utils/attachments.ts:1718-1724`

---

## 4. Memory Type Definitions

### 4.1 MemoryType Enum

```typescript
// src/utils/memory/types.ts
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
export type MemoryType = (typeof MEMORY_TYPES)[number]

// src/memdir/memoryTypes.ts
export const MEMORY_TYPE_VALUES = [
  'User',
  'Project',
  'Local',
  'Managed',
  'AutoMem',
  ...(feature('TEAMMEM') ? (['TeamMem'] as const) : []),
] as const
```

### 4.2 InstructionsMemoryType (CLAUDE.md types)

```typescript
type InstructionsMemoryType = 'User' | 'Project' | 'Local' | 'Managed'
```

---

## 5. Feature Flags

| Flag | Purpose |
|------|---------|
| `TEAMMEM` | Enable team memory (shared memory across team members) |
| `KAIROS` | Enable daily-log mode (append-only logging) |
| `EXTRACT_MEMORIES` | Background agent to extract memories |
| `tengu_coral_fern` | Enable past context search in memory |
| `tengu_moth_copse` | Skip MEMORY.md index (use topic files only) |

**Source**: `src/memdir/memdir.ts`, `src/memdir/paths.ts`

---

## 6. Memory File Size Limits

| Limit | Value | Purpose |
|-------|-------|---------|
| `MAX_ENTRYPOINT_LINES` | 200 | Max lines in MEMORY.md index |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | Max bytes in MEMORY.md |
| `MAX_MEMORY_CHARACTER_COUNT` | 40,000 | Recommended max for memory files |

**Source**: `src/memdir/memdir.ts:34-38`

---

## 7. Memory vs Other Persistence

The system prompt clarifies when to use memory vs other mechanisms:

```
## Memory and other forms of persistence

- When to use/update a PLAN instead of memory:
  → Non-trivial implementation tasks needing user alignment

- When to use/update TASKS instead of memory:
  → Breaking work into discrete steps within current conversation

- When to use MEMORY:
  → Information useful in FUTURE conversations
```

**Source**: `src/memdir/memdir.ts:254-258`

---

## 8. Memory Integrity: What NOT to Save

The prompt explicitly excludes:

```
## What NOT to save in memory

- Code patterns, conventions, architecture, file paths — derivable from code
- Git history, recent changes — use `git log` / `git blame`
- Debugging solutions — the fix is in code, commit message has context
- Anything in CLAUDE.md files
- Ephemeral task details (in-progress work, temporary state)
```

Even if user explicitly asks to save PR activity logs, the model is instructed to ask for the *surprising/non-obvious* part.

**Source**: `src/memdir/memoryTypes.ts:183-195`

---

## 9. Memory Drift Handling

Memories can become stale. The system instructs:

```
## Before recommending from memory

A memory naming a function/file/flag claims it existed WHEN memory was written.
It may have been renamed, removed, or never merged.

Before recommending it:
- If names a file path: check the file exists
- If names a function or flag: grep for it
- If user is about to act on it: verify first

"The memory says X exists" ≠ "X exists now"
```

**Source**: `src/memdir/memoryTypes.ts:240-256`

---

## 10. Source Code Index

### Core Memory Files

| File | Purpose |
|------|---------|
| `src/memdir/memdir.ts` | Main memory loading, `loadMemoryPrompt()`, `buildMemoryLines()` |
| `src/memdir/memoryTypes.ts` | Memory type definitions, frontmatter format, guidance text |
| `src/memdir/paths.ts` | Memory path resolution, `getAutoMemPath()` |
| `src/memdir/teamMemPrompts.ts` | Team memory prompt builder |
| `src/memdir/teamMemPaths.ts` | Team memory path resolution |

### CLAUDE.md System

| File | Purpose |
|------|---------|
| `src/utils/claudemd.ts` | `getMemoryFiles()`, file discovery, `@include` resolution |
| `src/utils/attachments.ts` | `memoryFilesToAttachments()`, `getNestedMemoryAttachmentsForFile()` |
| `src/utils/messages.ts` | `nested_memory` case rendering |

### Integration Points

| File | Line | Purpose |
|------|------|---------|
| `src/constants/prompts.ts` | 495 | System prompt memory section |
| `src/Tool.ts` | 215-222 | `nestedMemoryAttachmentTriggers`, `loadedNestedMemoryPaths` in context |

---

## 11. Summary: Memory Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Every API Call                                  │
│                                                                     │
│  System Prompt:                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ # Memory                                                     │  │
│  │ You have a persistent, file-based memory system at...       │  │
│  │                                                              │  │
│  │ ## Types of memory                                           │  │
│  │ ## How to save memories                                      │  │
│  │ ## Memory.md (index content only)                            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Message History:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ [Full conversation history]                                 │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Attachments (on-demand):                                          │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ nested_memory: CLAUDE.md content (when relevant)           │  │
│  │ relevant_memories: Topic file contents                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    On Tool Access                                   │
│                                                                     │
│  Model calls Read(file="src/auth/login.ts")                        │
│       ↓                                                             │
│  getNestedMemoryAttachmentsForFile("src/auth/login.ts")             │
│       ↓                                                             │
│  1. Check conditional rules with globs matching "src/auth/**"       │
│  2. Check CLAUDE.md in parent directories                           │
│  3. Check .claude/rules/*.md                                        │
│       ↓                                                             │
│  memoryFilesToAttachments() → add to request                       │
│       ↓                                                             │
│  Model sees relevant project instructions                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 12. Key Design Decisions

### 12.1 Why Two Memory Systems?

1. **Auto Memory (memdir)**: Typed, searchable, persistent across sessions
   - Designed for cross-conversation learning
   - Four-type taxonomy prevents memory bloat
   - MEMORY.md as index prevents token explosion

2. **CLAUDE.md**: Direct project instructions
   - Checked into version control (team shared)
   - Human-authored, not model-generated
   - Conditional rules for file-specific guidance

### 12.2 Why Attachments Instead of System Prompt?

- System prompt has size limits
- Attachments can be conditionally loaded based on context
- Different files can be loaded at different times
- LRU + dedup prevents infinite memory growth

### 12.3 Why Frontmatter Format?

- Enables automated processing
- Type field enables filtering
- Description enables relevance matching
- Structured format prevents drift

---

*Analysis based on Claude Code v2.1.88 source code*
