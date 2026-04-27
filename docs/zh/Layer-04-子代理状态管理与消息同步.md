# Layer 04: 子代理状态管理与消息同步

> 基于 Claude Code 源码架构分析 · 2026-04-24

## 1. 核心设计哲学：受控的隔离

在 Claude Code 中，子代理（Sub-agents / Forked Agents）被设计为在一个“受控沙箱”中运行。这种设计既要保证子代理能够感知主代理的环境，又要防止子代理的副作用污染全局状态。

---

## 2. 状态读取：快照继承 (Snapshot Inheritance)

当主代理创建子代理时，会通过 `createSubagentContext` 函数进行状态快照：

- **AppState 镜像**：子代理继承了主代理在启动那一刻的 `AppState`（如：当前权限模式、环境变量、已加载插件）。
- **文件记忆克隆**：使用 `cloneFileStateCache` 克隆主代理已读文件的状态，使子代理具备“先验知识”。
- **独立副本**：所有继承的状态均为副本或快照。子代理对这些状态的局部修改不会直接反映到主代理中。

---

## 3. 状态修改：宿主代行模式 (Host-Managed Update)

**核心结论：子代理本身没有修改主 `AppState` 的权限。**

子代理上下文中的 `setAppState` 通常被重写为 `no-op`（空函数）。那么子代理的消息是如何存入主 `AppState.tasks` 的？

### 3.1 监听者机制 (Observer Pattern)
消息的同步不是由子代理“主动推”，而是由主代理“主动拉”并记录：

1. **流式输出**：子代理作为异步生成器（`AsyncGenerator`），持续产出（`yield`）消息流。
2. **宿主拦截**：主代理的执行器（如 `runAgent.ts` 或 `LocalMainSessionTask.ts`）在外部消费这个流。
3. **特权写入**：执行器握有主代理的 **“真实 `setAppState` 权限”**。每当捕获到一条新的子代理消息，执行器会代为调用 `updateTaskState` 或 `setAppState`，将消息写入全局 `AppState.tasks`。

### 3.2 权限差异对比

| 操作类型 | 子代理内部调用 (`context.setAppState`) | 主代理执行器调用 (真实 `setAppState`) |
| :--- | :--- | :--- |
| **修改全局配置** | **被拦截 (无效/No-op)** | 不会执行此操作 |
| **写入对话历史** | **被拦截 (无效/No-op)** | **执行 (将消息同步至 tasks 分支)** |
| **任务状态更新** | 仅限 `setAppStateForTasks` (任务注册) | **执行 (更新状态、进度、耗时等)** |

---

## 4. 消息存储路径

子代理产生的消息（包括 `thinking`、`tool_use` 和 `text`）最终由宿主同步到 `AppState` 的如下位置：

```typescript
AppState: {
  tasks: {
    [taskId: string]: {
      type: 'local_agent',
      messages: Message[], // 存储子代理的完整对话历史
      progress: {
        toolUseCount: number,
        tokenCount: number
      }
    }
  }
}
```

- **活跃会话隔离**：主对话（Active Session）的消息为了性能存储在 `REPL.tsx` 的本地 State 中；而子代理消息为了持久化和可管理性，直接托管在全局 `AppState` 中。
- **UI 呈现**：当用户通过 `foregroundedTaskId` 切换视图时，系统从 `AppState.tasks` 中读取对应的 `messages` 数组进行渲染。

---

## 5. 特殊通道：任务管理

为了防止僵尸进程，系统允许子代理通过 `setAppStateForTasks` 修改 `AppState` 中的任务列表。这是唯一的例外，允许子代理向主代理报告其“存活状态”和“资源消耗”，从而在 UI 的底部状态栏或任务面板中显示进度。

---

## 6. 总结

子代理与主代理之间的 `AppState` 关系可以概括为：**“只读其面，不改其骨；身在沙箱，影投宿主”**。
- **只读其面**：子代理看得到主代理的状态快照。
- **不改其骨**：子代理无法修改主代理的全局设置。
- **身在沙箱**：子代理在隔离的环境副本中运行。
- **影投宿主**：子代理的所有行为足迹（消息流）由主代理这个“宿主”负责收集并同步到全局状态中心。

---
*基于 Claude Code v2.1.88 源代码分析*
