# 分叉代理 (Forked Agent) 重构设计

> **日期：** 2026-04-24  
> **主题：** 强化 `forked_agent.py` 以支持提示词缓存 (Prompt Cache) 优化与后台隔离。

## 1. 概述

目前 `server/agent/core/fork.py` 的实现过于简单。为了支持高性能的后台任务（如自动记忆提取、代码审计）并显著降低 Token 成本，我们需要引入严格的**沙箱隔离**、**权限静默化**与**缓存一致性**机制。

### 核心目标：
- **提示词缓存 (Prompt Cache) 效率**：确保上下文数据的确定性排序，最大化 LLM 的缓存命中率。
- **严格隔离与静默**：防止分叉代理修改全局 UI 状态或弹窗请求权限，中间过程记录于旁链（Sidechain）。
- **进程安全与健壮性**：支持优雅退出（Soft Abort）与实时 Token 审计。

---

## 2. 架构：分叉代理沙箱 (Sandbox)

### 2.1 组件隔离策略
| 组件 | 实现方式 | 目的 |
| :--- | :--- | :--- |
| **事件发射器** | `StubEventEmitter` | 覆盖 `emit` 为 no-op。后台任务产生的中间过程不干扰主 UI。 |
| **工具注册表** | `RestrictedRegistry` | 仅加载只读工具。输出 Schema 须**按字典序强制重排**以维持缓存哈希一致。 |
| **权限决策引擎** | `SilentPermissionPolicy` | 自动设为 `avoid_prompts=True`。若遇非授权操作直接报错/跳过，不弹窗阻塞。 |
| **文件状态缓存** | **写时复制 (CoW)** | 继承父代理已读文件列表（避免重复 I/O），但子代理的新读取记录仅存入 `local_patch`。 |
| **中止控制器** | `LinkedAbortController` | 链接至主会话。支持 **Soft Abort**：收到退出信号后预留 500ms 清理 I/O，而非硬杀进程。 |

### 2.2 状态记录与实时追踪
- **旁链转录 (Sidechain Transcript)**：中间过程（Thinking、工具调用）记录于 `task_logs`。
- **事务性 Token 审计**：通过回调机制**实时累加**子代理消耗。即使子代理因崩溃或超时被强杀，已产生的 Token 成本也必须准确记账。

---

## 3. 提示词缓存优化 (Cache Affinity)

### 3.1 缓存一致性的“五大支柱”
实现 `save_cache_safe_params()`，确保以下要素在字节级上与主代理最近一次 API 调用保持一致：
1. **System Prompt**：身份、思维协议及输出约束。
2. **User Context**：经过字典序排序并稳定渲染的 Markdown 文本（`json.dumps(sort_keys=True)`）。
3. **System Context**：稳定的 OS、Shell 及 CWD 信息。
4. **Tool Definitions**：工具池的 JSON Schema 必须按工具名称排序。
5. **Context Messages**：作为前缀注入的历史对话切片。

---

## 4. 指令引导 (Directive)

分叉代理将被包裹在约束标签中。**注入策略优化**：建议将 `FORK_DIRECTIVE` 注入到 User Message 的开头（`[Meta]` 标签），以提高 LLM 对约束指令的敏感度。

```markdown
<FORK_DIRECTIVE>
你是一个处于隔离环境的后台子进程，负责处理辅助任务（如审计或记忆提取）。
1. 你的输出仅供系统解析，严禁向用户提问，禁止输出前言或客套话。
2. 必须直接汇报结论。若遇到权限受限，请在日志中记录原因并返回当前最佳数据。
</FORK_DIRECTIVE>
```

---

## 5. 实施计划概览
1. **基础重构**：重命名 `fork.py` -> `forked_agent.py`。
2. **环境工厂**：实现 `create_subagent_context`，支持文件缓存的“写时复制”逻辑。
3. **缓存守护**：实现工具定义（Tools）和上下文（Context）的稳定序列化渲染引擎。
4. **鲁棒提取器**：实现 `extract_result_text()`，须防御性地剥离 `<thinking>` 标签并过滤所有 `tool_use` 内容，仅提取纯文本结论。
5. **集成适配**：在主循环 `agent.core.loop` 关键节点插入缓存参数捕获与实时 Token 归集钩子。
