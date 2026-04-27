# Claude Code 核心引擎：`QueryEngine` 技术详解

> 基于 Claude Code v2.1.88 源代码分析

## 1. 概述

`QueryEngine` 是 Claude Code 的**核心状态机与执行中枢**，位于 `src/QueryEngine.ts`。它承载了对话的完整生命周期，作为 CLI 层与底层 LLM 调用链路之间的执行桥梁。

其核心职责是：**在内存中维护会话一致性、执行工具流、管理 API 调用约束（预算、Turn 数）、实现多层上下文管理、并作为 Agent 智能行为的调度中心。**

---

## 2. 状态数据存储模型

`QueryEngine` 通过维护一组私有状态，确保了会话在长时间运行下的鲁棒性。

| 数据成员 | 类型 | 职责 |
| :--- | :--- | :--- |
| `mutableMessages` | `Message[]` | **核心上下文**：存储当前对话的所有历史消息，是 LLM 的输入源。 |
| `totalUsage` | `NonNullableUsage` | **成本会计**：实时累加 API 调用产生的 Token 消耗，用于预算管控。 |
| `readFileState` | `FileStateCache` | **文件一致性**：追踪会话中涉及的文件快照，防止读取冲突。 |
| `discoveredSkillNames`| `Set<string>` | **技能缓存**：追踪已发现的 Slash 命令与 Skill，提升匹配性能。 |
| `abortController` | `AbortController` | **中断控制**：支持用户随时挂起或中断正在运行的 Agent 任务。 |

---

## 3. 核心执行循环 (`submitMessage`)

`submitMessage` 实现了“请求 -> 思考 -> 执行 -> 反馈”的闭环，主要包含三个阶段：

### 3.1 上下文构建与审计
1. **权限审计**：通过 `toolPermissionContext` 对 LLM 的行为进行拦截，确保 AI 不会越权操作。
2. **提示词构建**：动态拼接 `System Prompt`，包含基础配置、动态记忆注入（AutoMem）以及用户的自定义附加指令。
3. **输入加工**：调用 `processUserInput` 处理 `/slash` 命令，将人类的自然语言转化为 LLM 可理解的指令集。

### 3.2 高效查询循环 (`query` Loop)
`QueryEngine` 驱动底层的 `query()` 生成器，处理关键业务逻辑：
*   **多维压缩联动**：协调 `query.ts` 产生的语义优化结果，确保上下文 Token 不超限。
*   **流式工具执行**：通过 `StreamingToolExecutor` 并行执行工具调用，实现任务的实时响应。
*   **容错与重试**：实现模型 Fallback 机制、API 错误分类与重试策略。

### 3.3 内存物理防御
*   **Boundary 确认**：当 LLM 返回包含 `compact_boundary` 的消息时，表示该历史段落已固化到磁盘，不再参与频繁运算。
*   **物理内存清理**：`QueryEngine` 调用 `splice` 将内存中的旧消息物理删除，将内存归还给操作系统。这一步对于长会话至关重要。

---

## 4. 关键设计哲学

### 双层级压缩策略 (Two-Layer Compaction)
为了解决上下文压力，`QueryEngine` 与 `query.ts` 形成了一种巧妙的分工机制：
1. **语义层（在 `query.ts` 中）**：执行 `snip`（去冗）、`microcompact`（内容精简）、`autocompact`（全文总结），优化 LLM 的上下文输入 Token。
2. **物理层（在 `QueryEngine.ts` 中）**：执行 `mutableMessages.splice()`。处理内存空间的物理占用，防止 RAM 随会话时长增长而无限膨胀。

### 鲁棒性与一致性保障
*   **异步 Transcript 写入**：`recordTranscript` 确保任何时候产生的消息都能实时同步到磁盘。即使在极端的工具调用崩溃中，也保证了对话的可恢复性。
*   **预算驱动的硬停止**：通过全局 `maxBudgetUsd` 监控，一旦成本超标，引擎会立即抛出异常并阻断后续的 API 交互，防止经济损失。

---

## 5. 总结：引擎角色定位

`QueryEngine` 不仅是一个请求包装器，它是 Claude Code 的**智能运行环境**。

它成功地将一个不稳定的、异步的 LLM 对话流，重构为一个**结构化、可审计、受控且具备自愈能力**的事务过程。无论是 CLI 的交互模式，还是 Headless 模式的自动化集成，`QueryEngine` 始终保证了对话历史的连续性与系统资源的安全。
