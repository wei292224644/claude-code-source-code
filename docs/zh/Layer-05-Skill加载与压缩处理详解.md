# Claude Code Skill 加载与压缩处理全流程分析

> 基于 Claude Code v2.1.88 源代码逻辑分析

## 1. 概述

在 Claude Code 架构中，Skill（技能）被设计为一种“持久化指令集”。由于 LLM 的上下文窗口有限，对话历史会定期触发压缩（Compaction）。为了确保模型在压缩后不会丢失已经激活的技能指令，系统实现了一套从 **内存追踪** 到 **自动附件注入** 的闭环机制。

---

## 2. 阶段一：Skill 的初始加载与注册

技能的生命周期始于模型调用 `activate_skill` 工具或用户执行斜杠命令（如 `/review`）。

### 2.1 核心调用链
1.  **触发**：模型在 `assistant` 消息中输出 `tool_use: activate_skill`。
2.  **执行**：`src/tools/SkillTool/SkillTool.ts` 中的 `call()` 函数被激活。
3.  **处理**：
    *   如果是本地技能，调用 `processPromptSlashCommand` 解析指令。
    *   如果是远程技能，调用 `executeRemoteSkill` 下载并处理 Markdown 内容。
4.  **状态写入**：调用 `src/bootstrap/state.ts` 中的 **`addInvokedSkill()`**。

### 2.2 状态变更 (State Changes)
*   **操作对象**：`src/bootstrap/state.ts` 中的 `STATE.invokedSkills` (类型为 `Map<string, InvokedSkillInfo>`)。
*   **写入内容**：
    *   `key`: `${agentId ?? ''}:${skillName}`（确保多代理环境下的隔离）。
    *   `value`: 包含技能名称、磁盘路径、**经过宏替换后的完整内容**以及激活时间戳。

---

## 3. 阶段二：对话运行中的状态保持

在正常的对话轮次中，Skill 内容已经作为之前的 `user` 消息（isMeta: true）存在于 `QueryEngine` 的 `mutableMessages` 数组中。模型可以通过对话历史感知到这些指令。

---

## 4. 阶段三：上下文压缩 (Compaction) 的处理

当上下文 Token 接近阈值时，`src/query.ts` 会触发 `autoCompact`。

### 4.1 核心函数：`compactConversation`
位于 `src/services/compact/compact.ts`。该函数负责将历史消息总结为摘要，并重建后续上下文。

### 4.2 技能恢复逻辑
在生成摘要消息后，系统会执行以下步骤：
1.  **调用 `createSkillAttachmentIfNeeded(agentId)`**：
    *   遍历内存中的 `STATE.invokedSkills`。
    *   按 `invokedAt` 时间戳进行**降序排列**（优先保留最近使用的技能）。
2.  **执行截断保护 (`truncateSkillContent`)**：
    *   单个技能内容上限：5,000 tokens (`POST_COMPACT_MAX_TOKENS_PER_SKILL`)。
    *   总预算上限：25,000 tokens (`POST_COMPACT_SKILLS_TOKEN_BUDGET`)。
    *   若超限，则保留头部（通常包含最重要的配置信息）并追加截断标记。
3.  **构建附件消息**：
    *   创建一个类型为 `invoked_skills` 的 **`AttachmentMessage`**。

### 4.3 状态流转图
```text
STATE.invokedSkills (内存)
      ↓
createSkillAttachmentIfNeeded()
      ↓
invoked_skills Attachment (消息对象)
      ↓
buildPostCompactMessages()
      ↓
注入到 QueryEngine.mutableMessages
```

---

## 5. 阶段四：压缩后的首轮 API 调用

压缩完成后，`QueryEngine` 的消息历史变成了：
1.  `compact_boundary` (系统消息)
2.  `summary` (模型生成的历史摘要)
3.  **`invoked_skills` (包含所有已加载技能副本的附件)**

在下一轮向 Anthropic API 发起请求时，系统会调用 `normalizeMessagesForAPI`。此时，`invoked_skills` 附件会被转化为一条 `isMeta: true` 的用户消息，将技能指令重新“喂”给模型。

---

## 6. 阶段五：会话恢复 (Resume) 时的处理

当用户关闭 CLI 再次使用 `/resume` 或 `resumeSession` 恢复对话时，内存中的 `STATE` 是空的。

### 6.1 核心函数：`restoreSkillStateFromMessages`
位于 `src/utils/conversationRecovery.ts`。

### 6.2 恢复流程
1.  系统读取 `.jsonl` 格式的磁盘日志。
2.  解析到类型为 `attachment` 且子类型为 `invoked_skills` 的消息。
3.  循环调用 **`addInvokedSkill()`**，将磁盘中的内容重新填入内存的 `STATE.invokedSkills`。
4.  **结果**：确保了如果恢复后的会话再次触发压缩，技能依然可以被正确地重新注入。

---

## 7. 关键细节补充

*   **幂等性**：在 `SkillTool` 激活技能前，会先调用 `getInvokedSkillsForAgent` 检查是否已存在。如果存在，则返回“已经激活”的提示，避免在历史中产生重复的技能指令块。
*   **清理机制**：当使用 `/clear` 命令时，会调用 `clearInvokedSkills()` 彻底排空内存中的技能映射，防止指令污染新会话。
*   **隔离性**：通过 `agentId` 严格区分主会话和子代理的技能，防止子代理加载的技能意外干扰主会话的逻辑。

---
*文档生成于：2026-05-08 · 基于源代码深度审计*
