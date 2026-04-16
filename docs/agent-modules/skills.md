# Skills 模块技术文档

> 本文档详细描述 Claude Code 中 Skills 模块的架构设计与实现细节。内容基于源代码分析，涵盖 SKILL.md 规范、解析器、加载流程、注册机制、条件触发及 Agent 集成。

**参考源码：**
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/skills/loadSkillsDir.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/skills/bundledSkills.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/utils/frontmatterParser.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/utils/argumentSubstitution.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/types/command.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/commands.ts`
- `/Users/wwj/Desktop/myself/claude-code-source-code/src/tools/SkillTool/SkillTool.ts`

---

## 1. Anthropic SKILL.md 规范详解

### 1.1 目录格式

Skill 采用**目录格式**存储，每个 Skill 对应一个同名目录，入口文件必须命名为 `SKILL.md`：

```
~/.claude/skills/          # user skills
.claude/skills/             # project skills
plugin/                     # plugin skills
<managed-path>/.claude/skills/  # policy-managed skills

# 示例结构
simplify/SKILL.md           # Skill 名称: simplify
update-config/SKILL.md      # Skill 名称: update-config
review-pr/SKILL.md          # Skill 名称: review-pr
```

> **注意：** Claude Code v2.1.x 仅支持 `skill-name/SKILL.md` 目录格式，不支持单个 `.md` 文件作为 Skill（这与 legacy `/commands/` 目录不同）。

### 1.2 Frontmatter 字段详解

SKILL.md 文件顶部包含 YAML frontmatter，格式为 `---` 包裹的 YAML 对象：

```yaml
---
name: simplify
description: Review changed code for reuse, quality, and efficiency
user-invocable: true
allowed-tools: [Read, Edit, Bash]
argument-hint: [focus-area]
arguments: focus-area
when-to-use: When you need to review code changes
model: sonnet
disable-model-invocation: false
context: inline
agent: general-purpose
effort: medium
paths: ["*.ts", "*.tsx", "src/**/*.{js,jsx}"]
hooks:
  PreToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: prettier --write $FILE
          timeout: 30
shell: bash
---
# Markdown body starts here
```

#### 核心字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `string` | Skill 的显示名称。若未提供，则使用目录名作为 `userFacingName()` 的回退值。 |
| `description` | `string \| null` | Skill 的描述，用于帮助文本和自动补全。若为 `null` 或空，则从 markdown body 首行提取。 |
| `user-invocable` | `boolean \| string` | 用户是否可通过 `/skill-name` 直接调用。默认 `true`。设为 `false` 则仅供模型通过 SkillTool 调用。 |
| `allowed-tools` | `string \| string[]` | Skill 执行时自动授权的工具列表（如 `["Read", "Bash"]`）。用户无需重复确认这些工具权限。 |
| `argument-hint` | `string \| null` | 参数提示文本，显示在命令后的灰色提示中（如 `[focus-area]`）。 |
| `arguments` | `string \| string[]` | 具名参数列表，解析后映射到 `$ARGUMENTS[n]` 替换。接受空格分隔字符串或 YAML 数组。 |
| `when-to-use` | `string \| null` | 详细的使用场景说明，帮助模型判断何时应触发该 Skill。 |
| `model` | `string \| null` | 模型别名或名称（如 `"sonnet"`, `"opus"`, `"haiku"`）。特殊值 `"inherit"` 表示使用父级模型。 |
| `disable-model-invocation` | `boolean \| string` | 若为 `true`，则禁止模型通过 SkillTool 调用此 Skill（用户仍可手动调用）。 |
| `context` | `"inline" \| "fork" \| null` | 执行上下文。`inline`（默认）将 Skill 内容展开到当前对话；`fork` 在独立子 Agent 中运行，有独立 token 预算。 |
| `agent` | `string \| null` | 当 `context: fork` 时指定使用的 Agent 类型（如 `"general-purpose"`, `"Bash"`）。 |
| `effort` | `string \| null` | Agent 的思考投入级别。有效值：`"low"`, `"medium"`, `"high"`, `"max"` 或正整数。控制模型的推理努力程度。 |
| `paths` | `string \| string[]` | 条件触发路径。Glob 模式数组，当模型操作匹配的文件时自动激活 Skill。使用与 CLAUDE.md 相同的 `ignore` 库进行匹配。 |
| `hooks` | `HooksSettings \| null` | Skill 调用时注册的 Hook 配置。Key 为 Hook 事件名，Value 为 matcher+hooks 数组。 |
| `shell` | `"bash" \| "powershell" \| null` | Skill 体内 `!` 代码块使用的 shell 类型。默认 `bash`。File-scoped — 由 Skill 作者指定，不读取用户配置。 |

#### 废弃/内部字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | `string \| null` | Skill 版本号（仅供参考）。 |
| `hide-from-slash-command-tool` | `string \| null` | **已废弃**，用于旧版 SlashCommand 工具的可见性控制。 |
| `skills` | `string \| null` | 预加载的 Skill 名称列表（仅适用于 Agents）。 |
| `type` | `string \| null` | Memory 类型（仅适用于 memory 文件）。 |

### 1.3 Markdown Body 中的变量替换

Skill 的 markdown body 支持以下变量替换，在 `getPromptForCommand()` 调用时实时展开：

| 变量 | 描述 |
|------|------|
| `${CLAUDE_SKILL_DIR}` | Skill 所在目录的绝对路径。Windows 上反斜杠统一转为正斜杠。 |
| `${CLAUDE_SESSION_ID}` | 当前会话 ID。 |
| `${ARGUMENTS}` | 完整参数字符串。 |
| `${ARGUMENTS[0]}`, `${ARGUMENTS[1]}`, ... | 按索引访问参数（0-indexed）。 |
| `${0}`, `${1}`, ... | `${ARGUMENTS[n]}` 的简写形式。 |
| `${argName}` | 具名参数（需在 `arguments` 字段中声明）。如 `arguments: foo bar`，则可用 `${foo}`, `${bar}`。 |

**替换示例（来自 `simplify.ts` bundled skill）：**

```typescript
// 源码中的 prompt 模板
const SIMPLIFY_PROMPT = `# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix any issues found.

// ...

## Additional Focus

${args}  // 这里直接拼接，无具名占位符
`

// 调用时
getPromptForCommand(args) {
  let prompt = SIMPLIFY_PROMPT
  if (args) {
    prompt += `\n\n## Additional Focus\n\n${args}`
  }
  return [{ type: 'text', text: prompt }]
}
```

### 1.4 Inline Shell 执行（!`...`）安全限制

Skill markdown body 中支持内联 shell 执行，语法为反引号包裹的命令：

```markdown
运行测试：`!npm test`
```

或代码块形式：

````markdown
```!
npm test && npm run build
```
````

**安全限制：**

1. **MCP Skills 禁用**：从 MCP 加载的 Skill（`loadedFrom === 'mcp'`）**永远不执行** inline shell 命令。这是因为 MCP Skill 是远程来源，内容不受本地信任。

2. **工具权限上下文**：`executeShellCommandsInPrompt()` 调用时传入 `toolPermissionContext`，其中 `alwaysAllowRules.command` 被设置为 Skill 的 `allowedTools`。这使得 Skill 内声明的工具自动获得执行权限。

3. **路径规范化**：Windows 上 `${CLAUDE_SKILL_DIR}` 中的反斜杠统一转为正斜杠，避免 shell 转义问题。

```typescript
// loadSkillsDir.ts 中的实现
if (baseDir) {
  const skillDir =
    process.platform === 'win32' ? baseDir.replace(/\\/g, '/') : baseDir
  finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
}

// 安全：MCP Skills 不执行 shell
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(
    finalContent,
    { ...toolUseContext, getAppState() { ... } },
    `/${skillName}`,
    shell,
  )
}
```

---

## 2. Skill 数据模型（TypeScript）

### 2.1 Command 类型层次

Claude Code 使用 TypeScript 类型系统描述 Skill/Command 的数据结构，位于 `/src/types/command.ts`：

```typescript
// 基类：所有命令的公共属性
type CommandBase = {
  availability?: CommandAvailability[]   // 可用性要求（'claude-ai' | 'console'）
  description: string                     // 描述
  hasUserSpecifiedDescription?: boolean  // 描述是否用户手动指定
  isEnabled?: () => boolean              // 条件启用检查
  isHidden?: boolean                    // 是否隐藏（不显示在帮助/自动补全中）
  name: string                           // 命令名
  aliases?: string[]                    // 别名（如 /h 等于 /help）
  isMcp?: boolean                       // 是否为 MCP 命令
  argumentHint?: string                  // 参数提示文本
  whenToUse?: string                     // 使用场景说明
  version?: string                       // 版本号
  disableModelInvocation?: boolean       // 是否禁止模型调用
  userInvocable?: boolean                // 是否用户可手动调用
  loadedFrom?: LoadedFrom                // 加载来源
  kind?: 'workflow'                      // 工作流类型标记
  immediate?: boolean                    // 是否立即执行（绕过队列）
  isSensitive?: boolean                  // 参数是否敏感（脱敏处理）
  userFacingName?: () => string          // 显示名称解析函数
}

// Prompt 类型命令（Skill 的主要形式）
type PromptCommand = {
  type: 'prompt'
  progressMessage: string                // 进度消息文本
  contentLength: number                  // 内容长度（用于 token 估算）
  argNames?: string[]                    // 参数名列表
  allowedTools?: string[]                // 允许的工具列表
  model?: string                         // 模型覆盖
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: {                          // 插件信息
    pluginManifest: PluginManifest
    repository: string
  }
  disableNonInteractive?: boolean
  hooks?: HooksSettings                  // Hook 配置
  skillRoot?: string                     // Skill 资源根目录
  context?: 'inline' | 'fork'            // 执行上下文
  agent?: string                          // Agent 类型（fork 模式）
  effort?: EffortValue                    // 思考投入级别
  paths?: string[]                       // 条件触发路径
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>        // 获取展开后的 prompt
}

// 完整 Command = Base + (Prompt | Local | LocalJSX)
type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)

// LoadedFrom 类型
type LoadedFrom =
  | 'commands_DEPRECATED'  // 旧版 /commands/ 目录
  | 'skills'               // 新版 /skills/ 目录
  | 'plugin'               // 插件提供
  | 'managed'             // Policy 管理
  | 'bundled'             // 内置 Skill
  | 'mcp'                 // MCP 服务器提供
```

### 2.2 Skill 的 source 字段语义

| source 值 | 含义 | 示例 |
|-----------|------|------|
| `bundled` | 内置 Skill，编译进 CLI 二进制 | `/simplify`, `/update-config` |
| `builtin` | 内置命令（非 Skill） | `/help`, `/clear` |
| `userSettings` | 用户级目录加载 | `~/.claude/skills/` |
| `projectSettings` | 项目级目录加载 | `.claude/skills/` |
| `policySettings` | Policy 管理目录 | `<managed-path>/.claude/skills/` |
| `plugin` | 插件提供 | Marketplace 插件 |
| `mcp` | MCP 服务器动态提供 | MCP 工具/提示 |
| `commands_DEPRECATED` | 旧版 `/commands/` 目录 | 兼容旧格式 |

### 2.3 getPromptForCommand 实现

`getPromptForCommand` 是 Skill 的核心方法，负责将 Skill 内容转换为 `ContentBlockParam[]`。实现位于 `loadSkillsDir.ts` 的 `createSkillCommand()` 工厂函数中：

```typescript
// loadSkillsDir.ts 第 344-399 行
async getPromptForCommand(args, toolUseContext) {
  // 1. 基础目录前缀（用于相对路径解析）
  let finalContent = baseDir
    ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
    : markdownContent

  // 2. 参数替换：$ARGUMENTS, $0, ${foo} 等
  finalContent = substituteArguments(
    finalContent,
    args,
    true,           // appendIfNoPlaceholder: 无占位符时追加 "ARGUMENTS: ..."
    argumentNames,  // 具名参数映射
  )

  // 3. ${CLAUDE_SKILL_DIR} 替换（Windows 路径规范化）
  if (baseDir) {
    const skillDir =
      process.platform === 'win32' ? baseDir.replace(/\\/g, '/') : baseDir
    finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
  }

  // 4. ${CLAUDE_SESSION_ID} 替换
  finalContent = finalContent.replace(
    /\$\{CLAUDE_SESSION_ID\}/g,
    getSessionId(),
  )

  // 5. Inline shell 执行（!`...` 语法，MCP Skills 除外）
  if (loadedFrom !== 'mcp') {
    finalContent = await executeShellCommandsInPrompt(
      finalContent,
      {
        ...toolUseContext,
        getAppState() {
          const appState = toolUseContext.getAppState()
          return {
            ...appState,
            toolPermissionContext: {
              ...appState.toolPermissionContext,
              alwaysAllowRules: {
                ...appState.toolPermissionContext.alwaysAllowRules,
                command: allowedTools,  // Skill 声明的工具自动放行
              },
            },
          }
        },
      },
      `/${skillName}`,
      shell,
    )
  }

  return [{ type: 'text', text: finalContent }]
}
```

---

## 3. Frontmatter 解析

### 3.1 解析流程

Frontmatter 解析位于 `/src/utils/frontmatterParser.ts`：

```typescript
// 正则匹配 ---...--- 块
const FRONTMATTER_REGEX = /^---\s*\n([\s\S]*?)---\s*\n?/

export function parseFrontmatter(
  markdown: string,
  sourcePath?: string,
): ParsedMarkdown {
  const match = markdown.match(FRONTMATTER_REGEX)

  if (!match) {
    // 无 frontmatter，直接返回原内容
    return { frontmatter: {}, content: markdown }
  }

  const frontmatterText = match[1] || ''
  const content = markdown.slice(match[0].length)

  let frontmatter: FrontmatterData = {}
  try {
    const parsed = parseYaml(frontmatterText) as FrontmatterData | null
    if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
      frontmatter = parsed
    }
  } catch {
    // YAML 解析失败，尝试 quoting 特殊字符后重试
    try {
      const quotedText = quoteProblematicValues(frontmatterText)
      const parsed = parseYaml(quotedText) as FrontmatterData | null
      if (parsed && typeof parsed === 'object' && !Array.isArray(parsed)) {
        frontmatter = parsed
      }
    } catch (retryError) {
      // 仍然失败，记录日志（不影响 Skill 加载）
      logForDebugging(
        `Failed to parse YAML frontmatter in ${sourcePath}: ${retryError}`,
        { level: 'warn' },
      )
    }
  }

  return { frontmatter, content }
}
```

### 3.2 FrontmatterData 类型定义

```typescript
export type FrontmatterData = {
  'allowed-tools'?: string | string[] | null
  description?: string | null
  type?: string | null                  // memory 类型
  'argument-hint'?: string | null
  when_to_use?: string | null
  version?: string | null
  'hide-from-slash-command-tool'?: string | null
  model?: string | null
  skills?: string | null                // Agent 预加载 skills
  'user-invocable'?: string | null
  hooks?: HooksSettings | null
  effort?: string | null
  context?: 'inline' | 'fork' | null
  agent?: string | null
  paths?: string | string[] | null       // 条件触发路径
  shell?: string | null
  [key: string]: unknown                 // 允许任意额外字段
}
```

### 3.3 特殊字符 quoting

YAML 中某些字符在未引号时会导致解析失败，frontmatterParser 会自动处理：

```typescript
// 特殊字符模式
const YAML_SPECIAL_CHARS = /[{}[\]*&#!|>%@`]|: /

function quoteProblematicValues(frontmatterText: string): string {
  // 匹配 key: value 行
  const match = line.match(/^([a-zA-Z_-]+):\s+(.+)$/)
  if (match) {
    const [, key, value] = match
    // 检测特殊字符，自动加双引号转义
    if (YAML_SPECIAL_CHARS.test(value)) {
      const escaped = value.replace(/\\/g, '\\\\').replace(/"/g, '\\"')
      return `${key}: "${escaped}"`
    }
  }
  return line
}
```

这允许 glob 模式如 `**\/*.{ts,tsx}` 正确解析。

### 3.4 错误处理策略

Frontmatter 解析错误采用**宽容策略**：

1. 第一次解析失败后，尝试 quoting 特殊字符后重试
2. 仍然失败时，仅记录 debug 日志，**不影响 Skill 加载**
3. 未提供的字段返回 `undefined`/`null`，调用方负责提供默认值

```typescript
// loadSkillsDir.ts 中的字段解析
const userInvocable =
  frontmatter['user-invocable'] === undefined
    ? true                                          // 默认值
    : parseBooleanFrontmatter(frontmatter['user-invocable'])  // "true" → true

const model =
  frontmatter.model === 'inherit'
    ? undefined
    : frontmatter.model
      ? parseUserSpecifiedModel(frontmatter.model as string)
      : undefined
```

---

## 4. Skill 加载流程

### 4.1 扫描路径优先级

Skill 加载按以下优先级（高优先级覆盖低优先级）：

```
bundled → builtin-plugin → managed → user → project → additional-dirs → legacy-commands
```

**具体路径：**

| 来源 | 路径 | source 值 |
|------|------|-----------|
| Policy 管理 | `<managed-path>/.claude/skills/` | `policySettings` |
| User | `~/.claude/skills/` | `userSettings` |
| Project | `.claude/skills/` (向上遍历到 HOME) | `projectSettings` |
| Additional dirs | `--add-dir` 指定的目录 | `projectSettings` |
| Legacy commands | `commands/` 目录 | `commands_DEPRECATED` |
| Bundled | 编译时注册（内存） | `bundled` |

### 4.2 完整加载实现

```typescript
// loadSkillsDir.ts 第 638-804 行（简化版）
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    // 1. 确定扫描路径
    const userSkillsDir = join(getClaudeConfigHomeDir(), 'skills')
    const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
    const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)

    // 2. --bare 模式：跳过自动发现，仅加载 --add-dir
    if (isBareMode()) {
      if (additionalDirs.length === 0) return []
      return (await Promise.all(
        additionalDirs.map(dir =>
          loadSkillsFromSkillsDir(join(dir, '.claude', 'skills'), 'projectSettings')
        )
      )).flat()
    }

    // 3. 并行加载所有来源
    const [
      managedSkills,      // policy settings
      userSkills,         // user settings
      projectSkillsNested, // project dirs
      additionalSkillsNested, // --add-dir
      legacyCommands,     // /commands/ backward compat
    ] = await Promise.all([
      loadSkillsFromSkillsDir(managedSkillsDir, 'policySettings'),
      loadSkillsFromSkillsDir(userSkillsDir, 'userSettings'),
      Promise.all(projectSkillsDirs.map(dir =>
        loadSkillsFromSkillsDir(dir, 'projectSettings')
      )),
      Promise.all(additionalDirs.map(dir =>
        loadSkillsFromSkillsDir(join(dir, '.claude', 'skills'), 'projectSettings')
      )),
      loadSkillsFromCommandsDir(cwd),
    ])

    // 4. 合并所有 skills
    const allSkillsWithPaths = [
      ...managedSkills,
      ...userSkills,
      ...projectSkillsNested.flat(),
      ...additionalSkillsNested.flat(),
      ...legacyCommands,
    ]

    // 5. 去重（按 realpath 解析，同一文件仅加载一次）
    const deduplicatedSkills = deduplicateByFileId(allSkillsWithPaths)

    // 6. 分离条件触发 skills 和无条件 skills
    const [unconditionalSkills, conditionalSkills] =
      partitionByConditionalFlag(deduplicatedSkills)

    // 7. 存储条件 skills，等待文件操作时激活
    for (const skill of conditionalSkills) {
      conditionalSkillsMap.set(skill.name, skill)
    }

    return unconditionalSkills
  }
)
```

### 4.3 单个目录的 Skill 加载

```typescript
// loadSkillsDir.ts 第 407-480 行
async function loadSkillsFromSkillsDir(
  basePath: string,
  source: SettingSource,
): Promise<SkillWithPath[]> {
  const fs = getFsImplementation()

  let entries
  try {
    entries = await fs.readdir(basePath)
  } catch (e) {
    if (!isFsInaccessible(e)) logError(e)
    return []
  }

  const results = await Promise.all(
    entries.map(async (entry): Promise<SkillWithPath | null> => {
      try {
        // 仅支持目录格式：skill-name/SKILL.md
        if (!entry.isDirectory() && !entry.isSymbolicLink()) {
          return null
        }

        const skillDirPath = join(basePath, entry.name)
        const skillFilePath = join(skillDirPath, 'SKILL.md')

        let content: string
        try {
          content = await fs.readFile(skillFilePath, { encoding: 'utf-8' })
        } catch (e) {
          if (!isENOENT(e)) {
            logForDebugging(`[skills] failed to read ${skillFilePath}: ${e}`, {
              level: 'warn',
            })
          }
          return null
        }

        // 解析 frontmatter 和 markdown body
        const { frontmatter, content: markdownContent } = parseFrontmatter(
          content,
          skillFilePath,
        )

        const skillName = entry.name
        const parsed = parseSkillFrontmatterFields(frontmatter, markdownContent, skillName)
        const paths = parseSkillPaths(frontmatter)

        return {
          skill: createSkillCommand({
            ...parsed,
            skillName,
            markdownContent,
            source,
            baseDir: skillDirPath,
            loadedFrom: 'skills',
            paths,
          }),
          filePath: skillFilePath,
        }
      } catch (error) {
        logError(error)
        return null
      }
    }),
  )

  return results.filter((r): r is SkillWithPath => r !== null)
}
```

### 4.4 去重机制

去重基于 `realpath()` 解析符号链接后的规范路径，防止同一文件通过不同路径访问时被重复加载：

```typescript
// loadSkillsDir.ts 第 726-763 行
const fileIds = await Promise.all(
  allSkillsWithPaths.map(({ skill, filePath }) =>
    skill.type === 'prompt'
      ? getFileIdentity(filePath)  // realpath 解析
      : Promise.resolve(null),
  ),
)

const seenFileIds = new Map<string, SettingSource | 'bundled' | 'mcp' | ...>()
for (const entry of allSkillsWithPaths) {
  const fileId = fileIds[i]
  if (fileId === null || fileId === undefined) {
    deduplicatedSkills.push(skill)
    continue
  }

  const existingSource = seenFileIds.get(fileId)
  if (existingSource !== undefined) {
    // 跳过重复：同文件已被其他 source 加载
    logForDebugging(
      `Skipping duplicate skill '${skill.name}' from ${skill.source} (same file already loaded from ${existingSource})`,
    )
    continue
  }

  seenFileIds.set(fileId, skill.source)
  deduplicatedSkills.push(skill)
}
```

### 4.5 动态 Skill 发现

对于项目目录下嵌套的 `.claude/skills/`（如 `src/.claude/skills/`），采用动态发现机制：

```typescript
// loadSkillsDir.ts 第 861-915 行
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string,
): Promise<string[]> {
  const fs = getFsImplementation()
  const resolvedCwd = cwd.endsWith(pathSep) ? cwd.slice(0, -1) : cwd
  const newDirs: string[] = []

  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)

    // 向上遍历到 cwd 但不包含 cwd 本身
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')

      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)
        try {
          await fs.stat(skillDir)
          // 检查 gitignore
          if (await isPathGitignored(currentDir, resolvedCwd)) {
            continue
          }
          newDirs.push(skillDir)
        } catch {
          // 目录不存在，正常
        }
      }

      const parent = dirname(currentDir)
      if (parent === currentDir) break
      currentDir = parent
    }
  }

  // 按深度排序（深的优先）
  return newDirs.sort(
    (a, b) => b.split(pathSep).length - a.split(pathSep).length,
  )
}
```

---

## 5. Skill 注册表

### 5.1 命令注册表架构

Claude Code 的命令注册采用**分层结构**：

```typescript
// commands.ts 中的命令注册
export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)

  // 获取动态发现的 skills（条件触发 + 动态加载）
  const dynamicSkills = getDynamicSkills()

  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_),
  )

  if (dynamicSkills.length === 0) {
    return baseCommands
  }

  // 合并动态 skills（去重）
  const baseCommandNames = new Set(baseCommands.map(c => c.name))
  const uniqueDynamicSkills = dynamicSkills.filter(
    s =>
      !baseCommandNames.has(s.name) &&
      meetsAvailabilityRequirement(s) &&
      isCommandEnabled(s),
  )

  return [...baseCommands, ...uniqueDynamicSkills]
}
```

### 5.2 命令查找

```typescript
// commands.ts
export function findCommand(
  name: string,
  commands: Command[],
): Command | undefined {
  return commands.find(cmd => {
    if (cmd.name === name) return true
    if (cmd.aliases?.includes(name)) return true
    return false
  })
}

export function hasCommand(
  name: string,
  commands: Command[],
): boolean {
  return findCommand(name, commands) !== undefined
}
```

### 5.3 SkillTool 中的命令查找

SkillTool 整合了所有命令来源（包括 MCP skills）：

```typescript
// SkillTool.ts 第 81-94 行
async function getAllCommands(context: ToolUseContext): Promise<Command[]> {
  const mcpSkills = context
    .getAppState()
    .mcp.commands.filter(
      cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp',
    )
  if (mcpSkills.length === 0) {
    return getCommands(getProjectRoot())
  }
  const localCommands = await getCommands(getProjectRoot())
  return uniqBy([...localCommands, ...mcpSkills], 'name')
}
```

### 5.4 动态 Skill 注册

运行时动态添加的 Skill（如条件触发 Skill）通过 `dynamicSkills` Map 管理：

```typescript
// loadSkillsDir.ts
const dynamicSkillDirs = new Set<string>()
const dynamicSkills = new Map<string, Command>()

export function getDynamicSkills(): Command[] {
  return Array.from(dynamicSkills.values())
}

export async function addSkillDirectories(dirs: string[]): Promise<void> {
  const loadedSkills = await Promise.all(
    dirs.map(dir => loadSkillsFromSkillsDir(dir, 'projectSettings')),
  )

  // 反序处理（浅层先加载，深层覆盖）
  for (let i = loadedSkills.length - 1; i >= 0; i--) {
    for (const { skill } of loadedSkills[i] ?? []) {
      if (skill.type === 'prompt') {
        dynamicSkills.set(skill.name, skill)
      }
    }
  }

  skillsLoaded.emit()  // 通知监听器
}
```

### 5.5 回调机制

Skill 加载完成后通过 Signal 通知其他模块（避免循环依赖）：

```typescript
const skillsLoaded = createSignal()

export function onDynamicSkillsLoaded(callback: () => void): () => void {
  return skillsLoaded.subscribe(() => {
    try {
      callback()
    } catch (error) {
      logError(error)
    }
  })
}
```

---

## 6. 条件触发（paths Frontmatter）

### 6.1 paths 字段格式

```yaml
---
name: my-skill
paths:
  - "*.ts"
  - "src/**/*.tsx"
  - "{foo,bar}/*.js"
---
```

支持逗号分隔和 YAML 数组两种格式，自动展开 brace patterns（如 `{a,b}` → `a`, `b`）。

### 6.2 条件 Skill 的生命周期

1. **加载时**：有 `paths` 的 Skill 被存入 `conditionalSkills` Map，**不返回**给主注册表
2. **文件操作时**：`activateConditionalSkillsForPaths()` 检查匹配
3. **匹配后**：Skill 移入 `dynamicSkills`，通知 `skillsLoaded`

```typescript
// loadSkillsDir.ts 第 771-796 行
// 分离条件和无条件 skills
for (const skill of deduplicatedSkills) {
  if (
    skill.type === 'prompt' &&
    skill.paths &&
    skill.paths.length > 0 &&
    !activatedConditionalSkillNames.has(skill.name)
  ) {
    newConditionalSkills.push(skill)
  } else {
    unconditionalSkills.push(skill)
  }
}

// 存储条件 skills
for (const skill of newConditionalSkills) {
  conditionalSkills.set(skill.name, skill)
}
```

### 6.3 激活逻辑

```typescript
// loadSkillsDir.ts 第 997-1058 行
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  if (conditionalSkills.size === 0) {
    return []
  }

  const activated: string[] = []

  for (const [name, skill] of conditionalSkills) {
    if (skill.type !== 'prompt' || !skill.paths || skill.paths.length === 0) {
      continue
    }

    // 使用 ignore 库进行 gitignore 风格匹配
    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      // 计算相对路径
      const relativePath = isAbsolute(filePath)
        ? relative(cwd, filePath)
        : filePath

      // 跳过无效路径
      if (
        !relativePath ||
        relativePath.startsWith('..') ||
        isAbsolute(relativePath)
      ) {
        continue
      }

      if (skillIgnore.ignores(relativePath)) {
        // 激活：移入 dynamicSkills
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        logForDebugging(
          `[skills] Activated conditional skill '${name}' (matched path: ${relativePath})`,
        )
        break
      }
    }
  }

  if (activated.length > 0) {
    skillsLoaded.emit()
  }

  return activated
}
```

### 6.4 调用位置

`activateConditionalSkillsForPaths` 在**文件操作相关的工具执行后**调用：

- `Read` 工具返回后
- `Write`/`Edit` 工具返回后
- `Glob`/`Grep` 工具返回后
- 其他可能访问新文件路径的工具

这样当模型读取或修改特定路径的文件时，相关的条件 Skill 自动激活。

---

## 7. Bundled Skills（内置 Skills）

### 7.1 内置 Skill 的结构

Bundled Skills 编译进 CLI 二进制，不依赖文件系统。定义位于 `src/skills/bundled/` 目录：

```
src/skills/bundled/
├── index.ts              # 统一注册入口
├── simplify.ts           # /simplify
├── updateConfig.ts       # /update-config
├── verify.ts             # /verify
├── debug.ts              # /debug
├── batch.ts              # /batch
├── loop.ts               # /loop（KAIROS feature）
├── remember.ts           # /remember
├── keybindings.ts        # /keybindings-help
├── claudeApi.ts          # /claude-api（BUILDING_CLAUDE_APPS feature）
├── mcpSkillBuilders.ts  # MCP skill builder 注册
└── ...                   # 更多内置 skills
```

### 7.2 注册机制

**步骤 1：定义 Skill**

```typescript
// src/skills/bundled/simplify.ts
const SIMPLIFY_PROMPT = `# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix any issues found.
...

export function registerSimplifySkill(): void {
  registerBundledSkill({
    name: 'simplify',
    description:
      'Review changed code for reuse, quality, and efficiency, then fix any issues found.',
    userInvocable: true,
    async getPromptForCommand(args) {
      let prompt = SIMPLIFY_PROMPT
      if (args) {
        prompt += `\n\n## Additional Focus\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

**步骤 2：在 index.ts 中注册**

```typescript
// src/skills/bundled/index.ts
export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()

  // Feature-gated skills
  if (feature('KAIROS') || feature('KAIROS_DREAM')) {
    const { registerDreamSkill } = require('./dream.js')
    registerDreamSkill()
  }
  if (feature('AGENT_TRIGGERS')) {
    const { registerLoopSkill } = require('./loop.js')
    registerLoopSkill()
  }
  if (feature('BUILDING_CLAUDE_APPS')) {
    const { registerClaudeApiSkill } = require('./claudeApi.js')
    registerClaudeApiSkill()
  }
  // ...
}
```

### 7.3 BundledSkillDefinition 类型

```typescript
// bundledSkills.ts
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  /**
   * Additional reference files to extract to disk on first invocation.
   * Keys are relative paths, values are content.
   * Skill prompt 会被前缀 "Base directory for this skill: <dir>"
   */
  files?: Record<string, string>
  getPromptForCommand: (
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

### 7.4 动态文件提取

Bundled Skill 可以附带参考文件，首次调用时提取到磁盘：

```typescript
// bundledSkills.ts 第 53-73 行
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition

  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      // 首次调用时提取文件
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }
  // ...
}
```

提取使用安全写入模式（`O_NOFOLLOW|O_EXCL`，0o600 权限），防止符号链接攻击。

### 7.5 添加新 Bundled Skill 的步骤

1. 在 `src/skills/bundled/` 创建新文件，如 `my-skill.ts`
2. 导出 `registerMySkillSkill(): void` 函数，调用 `registerBundledSkill()`
3. 在 `src/skills/bundled/index.ts` 中 `import` 并调用注册函数
4. 如有 feature gate，在 `initBundledSkills()` 中添加条件调用

---

## 8. Agent 中的 Skill 触发

### 8.1 触发检测流程

用户输入或模型决策需要调用 Skill 时，经过以下流程：

```
用户输入 "/simplify"
    ↓
processSlashCommand() 解析
    ↓
findCommand() 在命令注册表中查找
    ↓
getPromptForCommand() 获取展开后的 prompt
    ↓
ContentBlock → messages 注入
    ↓
模型处理 Skill 内容
```

### 8.2 SlashCommand 解析

```typescript
// processSlashCommand.tsx 第 309-395 行
export async function processSlashCommand(
  inputString: string,
  precedingInputBlocks: ContentBlockParam[],
  imageContentBlocks: ContentBlockParam[],
  attachmentMessages: AttachmentMessage[],
  context: ProcessUserInputContext,
  setToolJSX: SetToolJSXFn,
  uuid?: string,
  isAlreadyProcessing?: boolean,
  canUseTool?: CanUseToolFn,
): Promise<ProcessUserInputBaseResult> {
  const parsed = parseSlashCommand(inputString)
  if (!parsed) {
    // 非有效 slash 命令
    return { messages: [...], shouldQuery: true, ... }
  }

  const { commandName, args: parsedArgs, isMcp } = parsed

  // 验证命令存在
  if (!hasCommand(commandName, context.options.commands)) {
    // 未知命令处理...
  }

  // 获取命令详情并生成 messages
  const {
    messages: newMessages,
    shouldQuery: messageShouldQuery,
    allowedTools,
    model,
    effort,
    command: returnedCommand,
  } = await getMessagesForSlashCommand(
    commandName,
    parsedArgs,
    setToolJSX,
    context,
    precedingInputBlocks,
    imageContentBlocks,
    isAlreadyProcessing,
    canUseTool,
    uuid,
  )

  // ...
}
```

### 8.3 SkillTool 调用

模型通过 `SkillTool` 调用 Skill：

```typescript
// SkillTool.ts 第 580-648 行
async call({ skill, args }, context, canUseTool, parentMessage) {
  const commandName = skill.startsWith('/') ? skill.substring(1) : skill

  // 获取所有可用命令（包括 MCP skills）
  const commands = await getAllCommands(context)
  const command = findCommand(commandName, commands)

  // 记录使用
  recordSkillUsage(commandName)

  // Fork 模式：独立子 Agent
  if (command?.type === 'prompt' && command.context === 'fork') {
    return executeForkedSkill(command, commandName, args, context, canUseTool, parentMessage)
  }

  // Inline 模式：处理 slash 命令
  const { processPromptSlashCommand } = await import(
    'src/utils/processUserInput/processSlashCommand.js'
  )
  const processedCommand = await processPromptSlashCommand(
    commandName,
    args || '',
    commands,
    context,
  )

  // 返回新 messages 和 contextModifier
  return {
    data: { success: true, commandName, allowedTools, model },
    newMessages: processedCommand.messages,
    contextModifier(ctx) {
      // 修改 toolPermissionContext，注入 allowedTools
      // 修改 options.mainLoopModel，应用 model override
      // 修改 effortValue，应用 effort override
    },
  }
}
```

### 8.4 Inline Shell 执行安全链

```typescript
// executeShellCommandsInPrompt 位于 promptShellExecution.ts
// 流程：
// 1. 扫描 !`command` 和 ```! ... ``` 块
// 2. 通过 toolUseContext.getAppState() 获取当前权限上下文
// 3. 将 Skill 的 allowedTools 合并到 alwaysAllowRules.command
// 4. 执行 shell 命令
// 5. 替换原块为命令输出
```

### 8.5 多次 Skill 触发处理

当多个 Skill 的触发词匹配同一输入时：

1. **Slash 命令解析**：`parseSlashCommand` 仅解析第一个命令名（如 `/simplify review` → command=`simplify`, args=`review`）
2. **模型决策**：模型可自主决定调用多个 Skill（通过多次 SkillTool 调用）
3. **条件触发**：多个条件 Skill 可能同时匹配同一文件路径，均被激活

### 8.6 Hook 注册

Skill 的 hooks 在 `processSlashCommand` 执行时注册：

```typescript
// processSlashCommand.tsx 第 872-878 行
const hooksAllowedForThisSkill =
  !isRestrictedToPluginOnly('hooks') || isSourceAdminTrusted(command.source)
if (command.hooks && hooksAllowedForThisSkill) {
  const sessionId = getSessionId()
  registerSkillHooks(
    context.setAppState,
    sessionId,
    command.hooks,
    command.name,
    command.type === 'prompt' ? command.skillRoot : undefined,
  )
}
```

---

## 附录：完整字段默认值参考

| 字段 | 默认值 | 来源 |
|------|--------|------|
| `user-invocable` | `true` | `parseSkillFrontmatterFields` |
| `model` | `undefined`（继承父级） | 调用方处理 |
| `disableModelInvocation` | `false` | `parseBooleanFrontmatter` |
| `context` | `undefined`（inline） | `createSkillCommand` |
| `effort` | `undefined` | 调用方处理 |
| `shell` | `undefined`（bash） | `executeShellCommandsInPrompt` |
| `allowedTools` | `[]` | `createSkillCommand` |
| `paths` | `undefined` | `parseSkillPaths` |

---

## 附录：关键文件索引

| 文件 | 职责 |
|------|------|
| `src/skills/loadSkillsDir.ts` | Skill 目录扫描、加载、去重、条件触发 |
| `src/skills/bundledSkills.ts` | Bundled Skill 注册表、文件提取 |
| `src/skills/bundled/index.ts` | Bundled Skill 初始化入口 |
| `src/skills/bundled/simplify.ts` | /simplify 实现示例 |
| `src/skills/bundled/updateConfig.ts` | /update-config 实现示例 |
| `src/utils/frontmatterParser.ts` | Frontmatter 解析、paths 展开、shell 字段解析 |
| `src/utils/argumentSubstitution.ts` | `$ARGUMENTS`、`${foo}` 等变量替换 |
| `src/types/command.ts` | Command/PromptCommand 类型定义 |
| `src/commands.ts` | 命令注册表、getCommands、findCommand |
| `src/tools/SkillTool/SkillTool.ts` | SkillTool 实现、validateInput、checkPermissions |
| `src/utils/processUserInput/processSlashCommand.tsx` | Slash 命令处理、Hook 注册 |
| `src/utils/hooks/registerSkillHooks.ts` | Skill Hook 注册实现 |
| `src/utils/forkedAgent.ts` | Fork 模式子 Agent 上下文准备 |
| `src/services/mcp/client.ts` | MCP Skill 加载（type: 'prompt', loadedFrom: 'mcp'）|
