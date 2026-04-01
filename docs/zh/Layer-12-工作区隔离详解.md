# Layer 12: WORKTREE ISOLATION — 工作区隔离

## 概述

**WORKTREE ISOLATION** 是基于 Git Worktree 的隔离机制。代理可以在独立的工作区副本中工作，避免污染主仓库。位于 `src/utils/worktree.ts`。

---

## 源码位置

| 组件 | 路径 |
|------|------|
| 工作区核心 | `src/utils/worktree.ts` |
| 工作区模式开关 | `src/utils/worktreeModeEnabled.ts` |
| 获取工作区路径 | `src/utils/getWorktreePaths.ts` |
| 便携式路径获取 | `src/utils/getWorktreePathsPortable.ts` |
| Git 文件系统 | `src/utils/git/gitFilesystem.ts` |
| 进入工作区 | `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` |
| 退出工作区 | `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` |
| 会话存储 | `src/utils/sessionStorage.ts` |
| 钩子系统 | `src/utils/hooks.ts` |

---

## 工作区模式

```typescript
// src/utils/worktreeModeEnabled.ts:1-11
/**
 * Worktree mode is now unconditionally enabled for all users.
 */
export function isWorktreeModeEnabled(): boolean {
  return true
}
```

之前由 GrowthBook 标志 `tengu_worktree_mode` 控制，但因 `CACHED_MAY_BE_STALE` 模式在首次启动时返回 `false`，导致 `--worktree` 标志被静默忽略。

---

## 工作区数据结构

### WorktreeSession 类型

```typescript
// src/utils/worktree.ts:140-154
export type WorktreeSession = {
  originalCwd: string
  worktreePath: string
  worktreeName: string
  worktreeBranch?: string
  originalBranch?: string
  originalHeadCommit?: string
  sessionId: string
  tmuxSessionName?: string
  hookBased?: boolean
  creationDurationMs?: number
  usedSparsePaths?: boolean
}
```

### PersistedWorktreeSession 类型

```typescript
// src/types/logs.ts:149-159
export type PersistedWorktreeSession = {
  originalCwd: string
  worktreePath: string
  worktreeName: string
  worktreeBranch?: string
  originalBranch?: string
  originalHeadCommit?: string
  sessionId: string
  tmuxSessionName?: string
  hookBased?: boolean
}
```

### Slug 验证

```typescript
// src/utils/worktree.ts:66-87
export function validateWorktreeSlug(slug: string): void {
  if (slug.length > MAX_WORKTREE_SLUG_LENGTH) { ... }
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') { ... }  // 防止路径遍历
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) { ... }
  }
}
```

---

## 工作区路径计算

```typescript
// src/utils/worktree.ts:204-227
function worktreesDir(repoRoot: string): string {
  return join(repoRoot, '.claude', 'worktrees')
}

function worktreePathFor(repoRoot: string, slug: string): string {
  return join(worktreesDir(repoRoot), flattenSlug(slug))
}

export function worktreeBranchName(slug: string): string {
  return `worktree-${flattenSlug(slug)}`
}
```

### 工作区存储结构

```
<git-root>/
└── .claude/
    └── worktrees/
        ├── agent-a1b2c3d/     # 代理工作区
        ├── wf_12345678-abc/   # 工作流工作区
        └── ...
```

---

## 创建工作区

### createWorktreeForSession - 会话工作区创建

```typescript
// src/utils/worktree.ts:702-778
export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
  tmuxSessionName?: string,
  options?: { prNumber?: number },
): Promise<WorktreeSession>
```

此函数：
1. 首先验证 slug
2. **优先尝试钩子模式**（允许用户配置的 VCS）
3. **回退到 git worktree**（需要在 git 仓库中）
4. 执行创建后设置（设置传播、钩子、符号链接）

### createAgentWorktree - 代理工作区创建

```typescript
// src/utils/worktree.ts:902-952
export async function createAgentWorktree(slug: string): Promise<{
  worktreePath: string
  worktreeBranch?: string
  headCommit?: string
  gitRoot?: string
  hookBased?: boolean
}>
```

与会话工作区的关键区别：
- **不触及全局会话状态**（`currentWorktreeSession`、`process.chdir`、项目配置）
- 使用 `findCanonicalGitRoot` 始终定位到主仓库的 `.claude/worktrees/`
- 恢复时更新 mtime，防止过期工作区被清理

### 代理工具中的工作区创建

```typescript
// src/tools/AgentTool/AgentTool.tsx:590-593
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}
```

---

## 清理工作区

### cleanupWorktree - 会话工作区清理

```typescript
// src/utils/worktree.ts:813-894
export async function cleanupWorktree(): Promise<void>
```

执行：
1. 切换回 `originalCwd`
2. 钩子模式：委托给 `executeWorktreeRemoveHook`
3. Git 模式：运行 `git worktree remove --force`
4. 删除临时工作区分支
5. 清除会话状态并更新配置

### keepWorktree - 保留工作区

```typescript
// src/utils/worktree.ts:780-811
export async function keepWorktree(): Promise<void>
```

保留工作区但清除会话状态。

### cleanupStaleAgentWorktrees - 过期代理工作区清理

```typescript
// src/utils/worktree.ts:1058-1136
export async function cleanupStaleAgentWorktrees(
  cutoffDate: Date,
): Promise<number>
```

安全特性：
- **仅处理临时 slug 模式**（从不清理用户命名的工作区）：
  - `agent-a<7hex>` (AgentTool)
  - `wf_<8hex>-<3hex>` (WorkflowTool)
  - `bridge-<safeFilenameId>` (bridgeMain)
  - `job-<templateName>-<8hex>` (模板作业)
- 跳过当前会话的工作区
- **Fail-closed**：如果 `git status` 失败或显示跟踪的更改则跳过
- **Fail-closed**：如果任何提交不可从远程到达则跳过

### removeAgentWorktree - 移除代理工作区

```typescript
// src/utils/worktree.ts:961
export async function removeAgentWorktree(
  worktreePath: string,
  worktreeBranch?: string,
  gitRoot?: string,
): Promise<void>
```

---

## 代理工具中的工作区处理

### 工作区创建和清理流程

```typescript
// src/tools/AgentTool/AgentTool.tsx:644-685
const cleanupWorktreeIfNeeded = async (): Promise<{
  worktreePath?: string
  worktreeBranch?: string
}> => {
  if (!worktreeInfo) return {}

  if (hookBased) {
    // 钩子模式工作区总是保留，因为无法检测 VCS 更改
    return { worktreePath }
  }

  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit)
    if (!changed) {
      // 无更改，自动清理
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
      return {}
    }
  }
  // 有更改，保留并返回路径/分支给用户
  return { worktreePath, worktreeBranch }
}
```

**决策逻辑**：如果代理没有做出更改（新提交、未提交的更改），工作区自动清理。如果有更改，则保留并返回路径/分支给用户。

### hasWorktreeChanges 检查

```typescript
// src/utils/worktree.ts:1144
export async function hasWorktreeChanges(
  worktreePath: string,
  headCommit: string,
): Promise<boolean> {
  // 检查工作区是否有未提交的更改
}
```

---

## 工作区钩子（可扩展性）

### 钩子检查

```typescript
// src/utils/hooks.ts:4910-4920
export function hasWorktreeCreateHook(): boolean {
  const snapshotHooks = getHooksConfigFromSnapshot()?.['WorktreeCreate']
  if (snapshotHooks && snapshotHooks.length > 0) return true
  // ...
}
```

### 执行创建钩子

```typescript
// src/utils/hooks.ts:4928-4958
export async function executeWorktreeCreateHook(
  name: string,
): Promise<{ worktreePath: string }>
```

### 执行移除钩子

```typescript
// src/utils/hooks.ts:4967-5000
export async function executeWorktreeRemoveHook(
  worktreePath: string,
): Promise<boolean>
```

这些钩子允许除 git 之外的 VCS 系统管理工作区。

---

## Git 工作区路径检测

```typescript
// src/utils/getWorktreePaths.ts:18-70
export async function getWorktreePaths(cwd: string): Promise<string[]> {
  const { stdout, code } = await execFileNoThrowWithCwd(
    gitExe(),
    ['worktree', 'list', '--porcelain'],
    { cwd, preserveOutputOnError: false },
  )
  // 解析 "worktree <path>" 行
  const worktreePaths = stdout
    .split('\n')
    .filter(line => line.startsWith('worktree '))
    .map(line => line.slice('worktree '.length).normalize('NFC'))
  // 排序：当前工作区优先，然后按字母顺序
}
```

---

## Git 文件系统支持

```typescript
// src/utils/git/gitFilesystem.ts
// resolveGitDir - 解析实际的 .git 目录，处理 worktree 的 gitdir: 指针文件
// getCommonDir - 对于 worktree，返回主仓库的 .git 目录（共享的 refs）
// readWorktreeHeadSha - 快速路径读取 worktree 的 HEAD SHA，无需生成子进程
```

---

## tmux 集成

```typescript
// src/utils/worktree.ts:1180-1519
export async function execIntoTmuxWorktree(
  // --worktree --tmux 的快速路径
  // 创建工作区并 exec 到 tmux
  // 使用 new-session -d -s <name> -c <worktreePath>
)
```

iTerm2 control mode (`-CC`) 用于原生标签页集成。

---

## EnterWorktreeTool 和 ExitWorktreeTool

### EnterWorktreeTool

```typescript
// src/tools/EnterWorktreeTool/EnterWorktreeTool.ts
// 通过 createWorktreeForSession() 创建工作区
// 执行 process.chdir() 到工作区
// 持久化工作区状态到会话
```

### ExitWorktreeTool

```typescript
// src/tools/ExitWorktreeTool/ExitWorktreeTool.ts
// 删除前验证（检查未提交的更改）
// 支持 action: 'keep' 或 action: 'remove'
// 需要 discard_changes: true 才能删除有更改的工作区
```

---

## 工作区状态持久化

```typescript
// src/utils/sessionStorage.ts:2889-2920
export function saveWorktreeState(
  worktreeSession: PersistedWorktreeSession | null,
): void {
  const project = getProject()
  project.currentSessionWorktree = stripped
  if (project.sessionFile) {
    appendEntryToFile(project.sessionFile, {
      type: 'worktree-state',
      worktreeSession: stripped,
      sessionId: getSessionId(),
    })
  }
}
```

---

## 隔离模式工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                     工作区隔离流程                                   │
│                                                                     │
│  1. AgentTool(isolation: 'worktree')                                │
│     ↓                                                               │
│  2. createAgentWorktree(slug)                                      │
│     ↓                                                               │
│  3. Git worktree create → .claude/worktrees/agent-a1b2c3d/         │
│     ↓                                                               │
│  4. 代理在工作区中执行                                               │
│     - 独立的 Git 状态                                               │
│     - 不影响主仓库                                                  │
│     ↓                                                               │
│  5. hasWorktreeChanges() 检查                                       │
│     ↓                                                               │
│  6a. 无更改 → removeAgentWorktree() 清理                           │
│  6b. 有更改 → 保留工作区，返回路径给用户                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 关键设计

### 1. Fail-closed 安全机制

```typescript
// cleanupStaleAgentWorktrees 中
// 如果 git status 失败或显示更改，跳过清理
if (!changed) {
  await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
  return {}
}
return { worktreePath, worktreeBranch }  // 保留有更改的工作区
```

### 2. 钩子模式可扩展性

允许非 Git VCS（如 Mercurial、Svn）通过钩子管理工作区。

### 3. 临时 vs 用户命名工作区

```typescript
// 仅清理临时模式工作区
const EPHEMERAL_SLUG_PATTERNS = [
  /^agent-[a-f0-9]+$/,           // AgentTool
  /^wf_[a-f0-9]+-[a-f0-9]+$/,   // WorkflowTool
  /^bridge-/,                     // bridgeMain
  /^job-/,                        // 模板作业
]
```

### 4. mtime 防清理

```typescript
// createAgentWorktree 中
// 恢复时更新 mtime，防止过期工作区被清理
```

---

## 源码索引

| 功能 | 文件:行号 |
|------|----------|
| createWorktreeForSession | `src/utils/worktree.ts:702-778` |
| cleanupWorktree | `src/utils/worktree.ts:813-894` |
| keepWorktree | `src/utils/worktree.ts:780-811` |
| createAgentWorktree | `src/utils/worktree.ts:902-952` |
| removeAgentWorktree | `src/utils/worktree.ts:961` |
| cleanupStaleAgentWorktrees | `src/utils/worktree.ts:1058-1136` |
| hasWorktreeChanges | `src/utils/worktree.ts:1144` |
| validateWorktreeSlug | `src/utils/worktree.ts:66-87` |
| WorktreeSession | `src/utils/worktree.ts:140-154` |
| EnterWorktreeTool | `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` |
| ExitWorktreeTool | `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` |
| getWorktreePaths | `src/utils/getWorktreePaths.ts:18-70` |
| saveWorktreeState | `src/utils/sessionStorage.ts:2889-2920` |
| hasWorktreeCreateHook | `src/utils/hooks.ts:4910-4920` |
| executeWorktreeCreateHook | `src/utils/hooks.ts:4928-4958` |
| resolveGitDir | `src/utils/git/gitFilesystem.ts:40-76` |

---

*基于 Claude Code v2.1.88 源代码分析*
