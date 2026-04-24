# Layer 04: forkedAgent 逻辑详解

> 基于 Claude Code 源代码逻辑分析 · 2026-04-24

## 1. 概述

`src/utils/forkedAgent.ts` 是 Claude Code 架构中的“子代理隔离与环境初始化中心”。它的核心职责是为所有脱离主循环（Main Loop）运行的代理行为（如 Subagents、Teammates、后台自动化任务等）提供一个安全隔离、状态独立且高性能（高缓存命中率）的执行环境。

---

## 2. 核心功能模块

### 2.1 上下文隔离工厂：`createSubagentContext`

这是该文件最重要的函数。它通过克隆和覆盖（Stubbing）父代理的 `ToolUseContext`，为子代理制造一个“逻辑沙箱”。

#### 隔离策略：
- **状态修改拦截**：将 `setAppState` 设为空函数（no-op）。子代理在后台尝试修改全局 UI 状态（如切换主题、弹出全局通知）将被静默忽略，从而保护主界面的稳定性。
- **任务权限保留**：虽然禁用了通用的状态修改，但保留了 `setAppStateForTasks`。这确保了异步子代理在后台启动的 Bash 任务仍能正确注册到根存储中，防止产生“僵尸进程”。
- **文件状态克隆**：使用 `cloneFileStateCache` 克隆父代理的已读文件记录，使子代理继承“记忆”而不会在操作时污染父代理的缓存。
- **自动权限静默**：默认设置 `shouldAvoidPermissionPrompts: true`。子代理在后台运行时，如果遇到权限请求，系统会自动处理（通常是拒绝或基于规则通过），而不是弹窗阻塞用户。
- **生命周期绑定**：创建链接到父代理的 `AbortController`。如果主会话结束，所有关联的子代理会立即收到中止信号。

### 2.2 缓存守护者：`CacheSafeParams`

为了实现极致的性能和低成本，子代理必须命中 Anthropic API 的**提示词缓存（Prompt Cache）**。

#### 匹配要求：
该文件定义并管理 `CacheSafeParams`，确保以下五个核心参数在父子代理间字节级一致：
1. `systemPrompt` (系统提示词)
2. `userContext` (用户上下文环境)
3. `systemContext` (系统上下文环境)
4. `toolUseContext` (工具池定义)
5. `forkContextMessages` (历史对话前缀)

通过 `saveCacheSafeParams` 机制，系统可以捕获主代理最近一次成功的 API 调用参数，供后续的辅助任务（如 `/btw` 或记忆提取）复用。

### 2.3 派生循环执行器：`runForkedAgent`

这是专门为“后台静默任务”设计的轻量级执行引擎（例如用于 `session_memory` 提取）。

#### 执行流程：
1. **环境准备**：调用 `createSubagentContext` 建立隔离区。
2. **转录记录**：为任务生成独立的 `agentId`，并将过程记录到“旁链转录”（Sidechain Transcript）中。
3. **查询迭代**：直接调用底层的 `query()` 函数进入 LLM 迭代。
4. **指标统计**：累加所有 API 调用的 Token 消耗，并在结束时发送 `tengu_fork_agent_query` 遥测事件。

---

## 3. 业务逻辑集成

### 3.1 命令与技能预处理 (`prepareForkedCommandContext`)

当用户运行斜杠命令（如 `/review`）或触发技能时，该函数负责：
- **参数替换**：将技能模板中的 `$ARGUMENTS` 替换为实际输入。
- **动态授权**：通过 `createGetAppStateWithAllowedTools` 临时提升子代理的工具权限，确保它拥有执行该特定任务所需的工具（例如 `/commit` 命令需要允许使用 `git` 相关的工具）。

### 3.2 结果提取优化 (`extractResultText`)

由于代理对话可能包含复杂的思考过程（Thinking）和工具调用，该工具提供了一种标准化的方式，从最后一条助手消息中提取纯文本结论，作为任务的最终输出。

---

## 4. 关键设计模式

| 模式 | 说明 |
| :--- | :--- |
| **沙箱模式 (Sandbox)** | 通过覆盖敏感回调（如 `setAppState`）限制子代理的副作用范围。 |
| **原子上下文 (Isolated Context)** | 利用 `AsyncLocalStorage` 和克隆技术确保并行代理间的状态不冲突。 |
| **缓存亲和性 (Cache Affinity)** | 通过严格的参数校验和 `CacheSafeParams` 传递，最大化 API 性能。 |
| **失败闭合 (Fail-closed)** | 默认禁用 UI 交互和高风险回调，仅通过显式 `overrides` 开启。 |

---

## 5. 源码索引

| 符号 | 用途 |
| :--- | :--- |
| `createSubagentContext` | 创建隔离的工具执行上下文 |
| `runForkedAgent` | 运行一个完全独立的后台查询循环 |
| `CacheSafeParams` | 提示词缓存一致性参数集 |
| `prepareForkedCommandContext` | 为 Slash 命令准备执行环境 |
| `saveCacheSafeParams` | 持久化当前轮次的缓存参数供后续 Fork 使用 |

---
*基于 Claude Code v2.1.88 源代码分析*
