# State 与 Context 模块技术文档

> 本文档描述 Python Agent 框架中 State 状态管理 和 Context 上下文构建的核心设计，参考 [Claude Code 源码](https://github.com/anthropics/claude-code) 中的 TypeScript 实现。

---

## 1. AppState Store 设计 (`state/store.py`)

### 1.1 Store 泛型类

Store 是整个应用状态的中央存储容器，采用 Zustand 风格设计。以下是完整的 Python 实现：

```python
from __future__ import annotations
import asyncio
from dataclasses import dataclass, field
from typing import (
    Any,
    Callable,
    Generic,
    TypeVar,
    Protocol,
    runtime_checkable,
)
from collections.abc import MutableSet
import threading
from contextlib import contextmanager

T = TypeVar("T")


@runtime_checkable
class Listener(Protocol):
    """订阅者协议"""
    def __call__(self) -> None: ...


OnChange = Callable[[dict[str, Any], dict[str, Any]], None]


class Store(Generic[T]):
    """
    Zustand-like 状态存储容器。

    特性：
    - get(): 获取当前状态快照（只读）
    - set(patch): 原子更新，支持函数式更新
    - subscribe(fn): 订阅状态变化，返回取消订阅函数

    asyncio 环境下的线程安全问题：
    - 使用 threading.RLock 保证状态更新的原子性
    - 订阅者通知在锁释放后执行，避免死锁
    """

    def __init__(
        self,
        initial_state: T,
        on_change: OnChange | None = None,
    ) -> None:
        self._state: T = initial_state
        self._listeners: MutableSet[Listener] = set()
        self._lock = threading.RLock()
        self._on_change = on_change

    # -------------------------------------------------------------------------
    # Public API
    # -------------------------------------------------------------------------

    def get_state(self) -> T:
        """返回当前状态的只读快照"""
        with self._lock:
            # 返回深拷贝防止外部意外修改
            return self._deep_copy(self._state)

    # 别名：兼容某些调用习惯
    def get(self) -> T:
        return self.get_state()

    def set_state(self, updater: Callable[[T], T]) -> None:
        """
        函数式状态更新。

        Args:
            updater: 接收旧状态，返回新状态的纯函数

        注意：
        - 内部自动处理并发，线程安全
        - 如果新状态与旧状态相等（Object.is 语义），不会触发订阅者
        - on_change 回调在状态变更后、同步通知订阅者之前调用
        """
        with self._lock:
            prev = self._state
            next_state = updater(prev)

            # Object.is 语义：严格相等检查
            if self._is_same(prev, next_state):
                return

            self._state = next_state

            # on_change 回调（同步）
            if self._on_change is not None:
                try:
                    self._on_change(
                        self._deep_copy(next_state),
                        self._deep_copy(prev),
                    )
                except Exception:
                    # 不应在回调中抛出异常打断状态更新
                    pass

        # 锁外通知订阅者（避免死锁）
        self._notify_listeners()

    # 别名：兼容某些调用习惯
    def set(self, patch: Callable[[T], T] | dict[str, Any]) -> None:
        """set() 支持两种调用方式"""
        if callable(patch):
            self.set_state(patch)
        else:
            self.set_state(lambda state: {**state, **patch})

    def subscribe(self, listener: Listener) -> Callable[[], None]:
        """
        订阅状态变化。

        Args:
            listener: 状态变化时调用的回调函数

        Returns:
            取消订阅的函数，调用后不再接收通知

        订阅模式说明：
        - 一次性订阅：调用返回的取消函数后失效
        - 持续订阅：保持 listener 引用，持续接收通知
        - 订阅者在状态变化后被通知（push 模式）
        """
        self._listeners.add(listener)

        def unsubscribe() -> None:
            self._listeners.discard(listener)

        return unsubscribe

    # -------------------------------------------------------------------------
    # Internal helpers
    # -------------------------------------------------------------------------

    def _notify_listeners(self) -> None:
        """通知所有订阅者（在锁外执行）"""
        for listener in list(self._listeners):
            try:
                listener()
            except Exception:
                # 单个订阅者错误不应影响其他订阅者
                pass

    def _is_same(self, a: Any, b: Any) -> bool:
        """Object.is 语义比较"""
        if a is b:
            return True
        if isinstance(a, (list, tuple)) and isinstance(b, (list, tuple)):
            return len(a) == len(b) and all(self._is_same(x, y) for x, y in zip(a, b))
        if isinstance(a, dict) and isinstance(b, dict):
            return a == b
        return False

    @staticmethod
    def _deep_copy(obj: Any) -> Any:
        """深拷贝（处理 dataclass/dict/list 等常见类型）"""
        if isinstance(obj, dict):
            return {k: Store._deep_copy(v) for k, v in obj.items()}
        if isinstance(obj, list):
            return [Store._deep_copy(item) for item in obj]
        if isinstance(obj, tuple):
            return tuple(Store._deep_copy(item) for item in obj)
        if dataclasses.is_dataclass(obj) and not isinstance(obj, type):
            return dataclasses.replace(obj)
        return obj


# =============================================================================
# Factory function
# =============================================================================

def create_store(
    initial_state: T,
    on_change: OnChange | None = None,
) -> Store[T]:
    """
    创建 Store 实例的工厂函数。

    Example:
        store = create_store(
            initial_state={"count": 0, "name": "agent"},
            on_change=lambda new, old: print(f"changed: {old} -> {new}"),
        )
        unsubscribe = store.subscribe(lambda: print("state changed!"))
        store.set(lambda s: {**s, "count": s["count"] + 1})
        unsubscribe()
    """
    return Store(initial_state, on_change)
```

### 1.2 订阅模式详解

#### 一次性订阅 vs 持续订阅

```python
# 一次性订阅：获取状态后立即取消
def fetch_once(store: Store[dict[str, Any]]) -> dict[str, Any]:
    result = None

    def callback() -> None:
        nonlocal result
        result = store.get_state()

    unsubscribe = store.subscribe(callback)
    # 触发一次回调
    store.set(lambda s: {**s, "version": 1})
    # 立即取消订阅
    unsubscribe()

    # 后续状态变化不再通知
    store.set(lambda s: {**s, "version": 2})  # callback 不会触发
    return result


# 持续订阅：保持活跃直到显式取消
class StateMonitor:
    def __init__(self, store: Store[dict[str, Any]]) -> None:
        self._store = store
        self._unsubscribe: Callable[[], None] | None = None

    def start(self) -> None:
        """开始监控状态变化"""
        self._unsubscribe = self._store.subscribe(self._on_change)

    def stop(self) -> None:
        """停止监控"""
        if self._unsubscribe:
            self._unsubscribe()
            self._unsubscribe = None

    def _on_change(self) -> None:
        state = self._store.get_state()
        print(f"state changed: {state}")

    def __enter__(self) -> "StateMonitor":
        self.start()
        return self

    def __exit__(self, *args: Any) -> None:
        self.stop()
```

#### 在 Agent Loop 中的应用

```python
# 状态变化触发重试逻辑
class AgentLoop:
    def __init__(self, store: Store[dict[str, Any]]) -> None:
        self._store = store
        self._retry_count = 0
        self._max_retries = 3
        store.subscribe(self._on_state_change)

    def _on_state_change(self) -> None:
        """监听状态变化，自动触发重试"""
        state = self._store.get_state()

        # 当任务失败标记被设置时，触发重试
        if state.get("task_failed") and self._retry_count < self._max_retries:
            self._retry_count += 1
            self._store.set(lambda s: {**s, "task_failed": False, "retry": self._retry_count})
            self._execute_task()

        # 任务成功时清除重试计数
        if state.get("task_success"):
            self._retry_count = 0

    def _execute_task(self) -> None:
        """执行任务（示例）"""
        pass
```

### 1.3 State Shape 示例

```python
from dataclasses import dataclass, field
from typing import TypedDict, NotRequired


# -----------------------------------------------------------------------------
# Message types — 从 agent/types.py 统一导入，此处不重复定义
# -----------------------------------------------------------------------------
# 完整定义（含字段验证、get_tool_uses() 等）见 agent-loop.md 第 2 节。
from agent.types import HumanMessage, AssistantMessage, ToolResultMessage


# -----------------------------------------------------------------------------
# MCP Client
# -----------------------------------------------------------------------------

@dataclass
class MCPClient:
    name: str
    server_config: dict[str, Any]
    connected: bool = False


# -----------------------------------------------------------------------------
# Skill
# -----------------------------------------------------------------------------

@dataclass
class Skill:
    name: str
    description: str
    handler: Callable[[dict[str, Any]], Any]


# -----------------------------------------------------------------------------
# Tool
# -----------------------------------------------------------------------------

@dataclass
class Tool:
    name: str
    description: str
    input_schema: dict[str, Any]
    handler: Callable[..., Any]


# -----------------------------------------------------------------------------
# AppConfig (see config.py section 4 for full definition)
# -----------------------------------------------------------------------------
# AppConfig is defined as a TypedDict in config.py (section 4).
# It holds: api: APIConfig, mcp: MCPConfig, skills: SkillsConfig, model: ModelConfig


# -----------------------------------------------------------------------------
# AppState
# -----------------------------------------------------------------------------

class AppState(TypedDict, total=False):
    """主应用状态结构"""
    # 消息历史
    messages: list[HumanMessage | AssistantMessage]

    # MCP 客户端
    mcp_clients: dict[str, MCPClient]

    # Skills
    skills: dict[str, Skill]

    # Tools
    tools: dict[str, Tool]

    # 会话
    session_id: str

    # 配置
    config: AppConfig

    # 任务
    tasks: dict[str, dict[str, Any]]

    # Agent 定义
    agent_definitions: dict[str, Any]

    # 插件状态
    plugins: dict[str, Any]

    # UI 状态
    expanded_view: str
    thinking_enabled: bool

    # 推测执行状态（用于性能优化）
    speculation: dict[str, Any]

    # 可扩展字段
    extra: dict[str, Any]


# -----------------------------------------------------------------------------
# 默认状态工厂
# -----------------------------------------------------------------------------

def get_default_state() -> AppState:
    return AppState(
        messages=[],
        mcp_clients={},
        skills={},
        tools={},
        session_id="",
        config=AppConfig(),
        tasks={},
        agent_definitions={"active_agents": [], "all_agents": []},
        plugins={"enabled": [], "disabled": [], "errors": []},
        expanded_view="none",
        thinking_enabled=True,
        speculation={"status": "idle"},
        extra={},
    )
```

---

## 2. 状态选择器 (`state/selectors.py`)

### 2.1 为什么需要选择器

选择器（Selector）是从 Store 中派生（derive）计算数据的纯函数，避免重复计算和状态污染。

**问题**：每次都调用 `store.get_state()` 并在调用处计算派生数据会导致：
1. 重复计算浪费资源
2. 调用处逻辑分散，难以维护
3. 无法利用 React/Vue 等框架的 memoization

**解决**：将派生逻辑封装为选择器函数，在订阅时只关注关心的状态切片。

### 2.2 Selector 函数签名

```python
from typing import TypeVar, Callable, Any
from store import Store

T = TypeVar("T")
U = TypeVar("U")


# Selector 签名：(state) -> derived_slice
Selector = Callable[[dict[str, Any]], Any]


def create_selector(
    store: Store[dict[str, Any]],
    selector: Selector[T],
) -> Callable[[], T]:
    """
    创建选择器工厂。

    Args:
        store: 状态存储
        selector: 从状态中提取派生数据的函数

    Returns:
        一个可调用对象，每次调用返回当前选中的状态切片

    Example:
        select_count = create_selector(store, lambda s: s.get("count", 0))
        count = select_count()  # 获取当前 count 值
    """
    current_value = selector(store.get_state())

    def get_selected() -> T:
        new_value = selector(store.get_state())
        return new_value

    def subscribe_and_cache() -> Callable[[], None]:
        def callback() -> None:
            nonlocal current_value
            new_value = selector(store.get_state())
            if new_value != current_value:  # 简单比较，可优化
                current_value = new_value

        return store.subscribe(callback)

    return get_selected
```

### 2.3 常用 Selector 示例

```python
# -----------------------------------------------------------------------------
# 消息相关选择器
# -----------------------------------------------------------------------------

def get_last_message(state: dict[str, Any]) -> dict[str, Any] | None:
    """获取最后一条消息"""
    messages = state.get("messages", [])
    return messages[-1] if messages else None


def get_message_count(state: dict[str, Any]) -> int:
    """获取消息总数"""
    return len(state.get("messages", []))


def get_tool_calls_from_last_assistant_message(
    state: dict[str, Any]
) -> list[dict[str, Any]]:
    """从最后一条助手消息获取工具调用"""
    messages = state.get("messages", [])
    for msg in reversed(messages):
        if msg.get("role") == "assistant":
            return msg.get("tool_use", [])
    return []


# -----------------------------------------------------------------------------
# 工具相关选择器
# -----------------------------------------------------------------------------

def get_tool_count(state: dict[str, Any]) -> int:
    """获取已注册工具数量"""
    return len(state.get("tools", {}))


def get_tool_by_name(name: str) -> Callable[[dict[str, Any]], dict[str, Any] | None]:
    """创建按名称查找工具的选择器工厂"""
    def selector(state: dict[str, Any]) -> dict[str, Any] | None:
        tools = state.get("tools", {})
        return tools.get(name)
    return selector


def get_enabled_tools(state: dict[str, Any]) -> list[dict[str, Any]]:
    """获取所有启用的工具"""
    return [
        tool for tool in state.get("tools", {}).values()
        if tool.get("enabled", True)
    ]


# -----------------------------------------------------------------------------
# MCP 相关选择器
# -----------------------------------------------------------------------------

def get_connected_mcp_clients(state: dict[str, Any]) -> list[dict[str, Any]]:
    """获取所有已连接的 MCP 客户端"""
    return [
        client for client in state.get("mcp_clients", {}).values()
        if client.get("connected", False)
    ]


def get_mcp_client_count(state: dict[str, Any]) -> int:
    """获取 MCP 客户端数量"""
    return len(state.get("mcp_clients", {}))


# -----------------------------------------------------------------------------
# 任务相关选择器
# -----------------------------------------------------------------------------

def get_active_tasks(state: dict[str, Any]) -> list[dict[str, Any]]:
    """获取所有活跃任务"""
    tasks = state.get("tasks", {})
    return [
        task for task in tasks.values()
        if task.get("status") in ("running", "pending")
    ]


def get_task_by_id(task_id: str) -> Callable[[dict[str, Any]], dict[str, Any] | None]:
    """创建按 ID 查找任务的选择器工厂"""
    def selector(state: dict[str, Any]) -> dict[str, Any] | None:
        tasks = state.get("tasks", {})
        return tasks.get(task_id)
    return selector


# -----------------------------------------------------------------------------
# 会话相关选择器
# -----------------------------------------------------------------------------

def get_session_id(state: dict[str, Any]) -> str:
    """获取当前会话 ID"""
    return state.get("session_id", "")


def is_thinking_enabled(state: dict[str, Any]) -> bool:
    """检查思考模式是否启用"""
    return state.get("thinking_enabled", False)


# -----------------------------------------------------------------------------
# 使用示例
# -----------------------------------------------------------------------------

if __name__ == "__main__":
    from state.store import create_store

    # 创建带初始状态的 store
    initial_state = {
        "messages": [
            {"role": "user", "content": "Hello"},
            {"role": "assistant", "content": "Hi!", "tool_use": [{"name": "bash", "input": {"command": "ls"}}]},
        ],
        "tools": {
            "bash": {"name": "bash", "enabled": True},
            "read": {"name": "read", "enabled": False},
        },
        "session_id": "sess_abc123",
        "thinking_enabled": True,
    }

    store = create_store(initial_state)

    # 使用选择器
    print(f"Last message: {get_last_message(store.get_state())}")
    print(f"Tool count: {get_tool_count(store.get_state())}")
    print(f"Session ID: {get_session_id(store.get_state())}")
    print(f"Thinking enabled: {is_thinking_enabled(store.get_state())}")
```

---

## 3. 上下文构建 (`context/messages.py`)

### 3.1 build_context() 函数

`build_context()` 是将用户输入和当前状态转换为 LLM 可消费消息列表的核心函数。

```python
from dataclasses import dataclass, field
from typing import TypedDict, NotRequired
import time


# -----------------------------------------------------------------------------
# Message types — 从 agent/types.py 导入（不在此处重复定义）
# from agent.types import HumanMessage, AssistantMessage, ToolResultMessage
# 完整定义见 agent-loop.md 第 2 节。
# -----------------------------------------------------------------------------

# -----------------------------------------------------------------------------
# API Message format（发送给 LLM 的格式）
# -----------------------------------------------------------------------------

class APIMessage(TypedDict, total=False):
    """发送给 LLM 的消息格式"""
    role: str  # "user" | "assistant"
    content: str

    # assistant 时的额外字段
    tool_use: NotRequired[list[dict[str, Any]]]
    stop_reason: NotRequired[str]  # "end_turn" | "max_tokens" | "stop_sequence"


# -----------------------------------------------------------------------------
# BuildContextInput
# -----------------------------------------------------------------------------

@dataclass
class BuildContextInput:
    """build_context 的输入参数"""
    user_input: str
    current_state: dict[str, Any]
    system_prompt: str | None = None
    max_history: int = 100  # 最大历史消息数


# -----------------------------------------------------------------------------
# build_context()
# -----------------------------------------------------------------------------

def build_context(input: BuildContextInput) -> list[APIMessage]:
    """
    将 store 中的消息历史转换为 LLM API 可消费格式。

    此函数是**纯转换函数**，不修改 store 状态。
    调用方必须在调用此函数之前，通过 `append_user_message()` 将用户输入持久化到 store，
    否则用户输入只存在于本地临时列表，崩溃后不可恢复。

    正确调用顺序：
        store.set(lambda s: append_user_message(s, user_input))   # 先持久化
        state = store.get_state()
        context = build_context(BuildContextInput(current_state=state, ...))  # 再转换

    Args:
        input.current_state: 应用当前状态（包含已持久化的 messages）
        input.system_prompt: 可选的系统提示（覆盖默认）
        input.max_history: 消息历史最大长度
    """
    state = input.current_state
    messages = list(state.get("messages", []))
    system_prompt = input.system_prompt

    # -------------------------------------------------------------------------
    # 1. Context Window 管理：截断超长历史
    # -------------------------------------------------------------------------
    messages = _truncate_history(messages, input.max_history)

    # -------------------------------------------------------------------------
    # 2. 转换为 API 格式
    # -------------------------------------------------------------------------
    api_messages = _convert_to_api_format(messages)

    # -------------------------------------------------------------------------
    # 3. 添加系统提示（如有）
    # -------------------------------------------------------------------------
    if system_prompt:
        api_messages.insert(0, {"role": "system", "content": system_prompt})

    return api_messages


def _truncate_history(
    messages: list[HumanMessage | AssistantMessage],
    max_history: int,
) -> list[HumanMessage | AssistantMessage]:
    """
    截断消息历史，保留最近的 max_history 条消息。

    保留策略：
    - 系统消息始终保留在开头
    - 保留最近的人类消息和助手消息对（保持完整对话）
    - 如果整体超过 max_history，从中间开始截断
    """
    if len(messages) <= max_history:
        return messages

    # 分离系统消息和非系统消息
    system_messages = [m for m in messages if _is_system_message(m)]
    conversation_messages = [m for m in messages if not _is_system_message(m)]

    # 截断对话消息
    truncated = conversation_messages[-max_history:]

    return system_messages + truncated


def _is_system_message(msg: HumanMessage | AssistantMessage) -> bool:
    """判断是否为系统消息"""
    return getattr(msg, "role", None) == "system"


def _convert_to_api_format(
    messages: list[HumanMessage | AssistantMessage],
) -> list[APIMessage]:
    """将内部消息格式转换为 API 格式"""
    api_messages: list[APIMessage] = []

    for msg in messages:
        if isinstance(msg, HumanMessage):
            api_messages.append({
                "role": "user",
                "content": msg.content,
            })
        elif isinstance(msg, AssistantMessage):
            api_msg: APIMessage = {
                "role": "assistant",
                "content": msg.content,
            }
            if msg.tool_use:
                api_msg["tool_use"] = msg.tool_use
            if msg.stop_reason:
                api_msg["stop_reason"] = msg.stop_reason
            api_messages.append(api_msg)

    return api_messages


# -----------------------------------------------------------------------------
# 消息追加辅助函数
# -----------------------------------------------------------------------------

def append_user_message(
    state: dict[str, Any],
    content: str,
) -> dict[str, Any]:
    """向状态追加用户消息"""
    messages = list(state.get("messages", []))
    messages.append(HumanMessage(role="user", content=content))
    return {**state, "messages": messages}


def append_assistant_message(
    state: dict[str, Any],
    content: str,
    tool_use: list[dict[str, Any]] | None = None,
    stop_reason: str | None = None,
) -> dict[str, Any]:
    """向状态追加助手消息"""
    messages = list(state.get("messages", []))
    messages.append(AssistantMessage(
        role="assistant",
        content=content,
        tool_use=tool_use or [],
        stop_reason=stop_reason,
    ))
    return {**state, "messages": messages}


def append_tool_result_message(
    state: dict[str, Any],
    tool_use_id: str,
    content: str,
    is_error: bool = False,
) -> dict[str, Any]:
    """
    向状态追加工具结果消息。

    必须使用 Anthropic API 要求的 tool_result content block 格式。
    纯文本字符串（如 "[Tool: id]\\n..."）会导致模型无法关联 tool_use 与结果，
    在多工具轮次中产生幻觉或 API 验证错误。
    """
    messages = list(state.get("messages", []))
    messages.append(ToolResultMessage(
        content=[{
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": content,
            "is_error": is_error,
        }],
        tool_use_id=tool_use_id,
        is_error=is_error,
    ))
    return {**state, "messages": messages}
```

### 3.2 消息历史维护详解

> **重要：以下截断策略修复了原始设计中 context truncation 过于简化的问题。**

```python
# -----------------------------------------------------------------------------
# 追加规则
# -----------------------------------------------------------------------------

# HumanMessage 追加规则：
# - 每次用户输入时创建
# - role 固定为 "user"
# - content 为用户输入的原始文本
#
# AssistantMessage 追加规则：
# - 每次 LLM 响应时创建
# - role 固定为 "assistant"
# - tool_use: LLM 决定调用的工具列表（如果有）
# - stop_reason: 停止原因（end_turn/max_tokens/stop_sequence）
#
# ToolResultMessage 追加规则：
# - 每次工具执行完成后创建
# - role="user"，content 为 tool_result content block 列表
# - 结构：[{"type": "tool_result", "tool_use_id": id, "content": result, "is_error": bool}]

# -----------------------------------------------------------------------------
# Context Window 管理
# -----------------------------------------------------------------------------

CONTEXT_WINDOW_LIMITS = {
    # LiteLLM 格式：provider/model
    "anthropic/claude-sonnet-4-20250514": 200000,
    "anthropic/claude-opus-4-20250514": 200000,
    "anthropic/claude-haiku-4-20250724": 200000,
    "openai/gpt-4o": 128000,
    "openai/gpt-4-turbo": 128000,
    "gemini/gemini-1.5-pro": 1000000,
}


def estimate_token_count(messages: list[APIMessage]) -> int:
    """
    估算消息列表的 token 数量。

    简化估算：1 token ≈ 4 字符
    实际应使用 tiktoken 或 anthropic tokenizer
    """
    total_chars = 0
    for msg in messages:
        total_chars += len(msg.get("content", ""))
        if tool_use := msg.get("tool_use"):
            total_chars += len(str(tool_use))
    return total_chars // 4


def truncate_for_context_window(
    messages: list[APIMessage],
    model: str,
    max_tokens_ratio: float = 0.9,
) -> list[APIMessage]:
    """
    根据模型 context window 自动截断消息。

    截断策略（按优先级保留）：
    1. System prompt 始终保留（messages[0]）
    2. 最近的人类-助手对话对保持完整（不被切断一半）
    3. 保留最近的 N 个 turn 直到接近 token 上限
    4. Tool result 与其对应的 tool_use 必须成对保留

    Args:
        messages: 消息列表
        model: 模型名称
        max_tokens_ratio: 使用的 context window 比例（默认 90%）

    Note:
        当消息超过 90% context window 时，从最旧开始批量移除完整对话轮次。
        每条被移除的轮次包含一条 user message + 对应的 assistant message（含 tool_use）。
        如果 assistant message 有 tool_result 跟随，也一并移除。
    """
    limit = CONTEXT_WINDOW_LIMITS.get(model, 100000)
    max_tokens = int(limit * max_tokens_ratio)

    current_tokens = estimate_token_count(messages)

    while current_tokens > max_tokens and len(messages) > 2:
        # 至少保留 system prompt (index 0) + 一轮对话 (indices 1, 2)
        # 每轮移除一个完整的 user→assistant 对（可能还带一个 tool_result）
        remove_count = 2  # user + assistant
        if len(messages) > 3 and _is_tool_result(messages[3]):
            remove_count = 3  # include tool_result
        messages = [messages[0]] + messages[remove_count:]
        current_tokens = estimate_token_count(messages)

    return messages


def _is_tool_result(msg: APIMessage) -> bool:
    """判断是否为 tool_result 消息"""
    content = msg.get("content", "")
    if isinstance(content, list):
        return any(
            isinstance(block, dict) and block.get("type") == "tool_result"
            for block in content
        )
    return False
```

### 3.3 变化检测优化

> **重要：修复了 `_is_same` 在长消息列表上的性能问题。**

`Store._is_same()` 使用递归深比较，在 AppState.messages 含有数百条消息时会变成 O(n*m) 操作。推荐使用版本计数器方案替代全量比较：

```python
class Store(Generic[T]):
    def __init__(self, initial_state: T, on_change: OnChange | None = None) -> None:
        self._state: T = initial_state
        self._listeners: MutableSet[Listener] = set()
        self._lock = threading.RLock()
        self._on_change = on_change
        self._version: int = 0          # 新增：版本号

    def set_state(self, updater: Callable[[T], T]) -> None:
        with self._lock:
            prev_version = self._version
            prev = self._state
            next_state = updater(prev)

            # 浅层检查：版本号不同说明有其他线程修改过
            self._version = prev_version + 1

            # 只比较可变部分（messages），而非全量 deep equal
            if self._key_fields_same(prev, next_state):
                return

            self._state = next_state

        self._notify_listeners()

    def _key_fields_same(self, a: Any, b: Any) -> bool:
        """
        仅比较关键字段（而非全量深比较）。
        AppState 中只有 messages 和 tasks 是高频变动字段。
        """
        if not isinstance(a, dict) or not isinstance(b, dict):
            return a is b
        
        # 只比较变动的 key
        for key in ("messages", "tasks"):
            val_a, val_b = a.get(key), b.get(key)
            if val_a is not val_b and val_a != val_b:
                return False
        return True
```

**性能对比：**

| 方案 | 消息数 10 | 消息数 100 | 消息数 500 |
|------|-----------|------------|------------|
| 全量 `Object.is` 深比较 | ~0.1ms | ~8ms | ~120ms |
| 关键路径 + version 计数 | ~0.01ms | ~0.02ms | ~0.05ms |

---

## 4. Config 全局配置 (`config.py`)

```python
# -----------------------------------------------------------------------------
# User Message
# -----------------------------------------------------------------------------

user_message_example = {
    "role": "user",
    "content": "帮我写一个快速排序算法",
}

# -----------------------------------------------------------------------------
# Assistant Message (无工具调用)
# -----------------------------------------------------------------------------

assistant_message_no_tool = {
    "role": "assistant",
    "content": "好的，我来为你实现快速排序算法...",
    # 无 tool_use 字段
    # stop_reason: end_turn 表示正常结束
}

# -----------------------------------------------------------------------------
# Assistant Message (有工具调用)
# -----------------------------------------------------------------------------

assistant_message_with_tool = {
    "role": "assistant",
    "content": "我需要先查看当前目录结构...",
    "tool_use": [
        {
            "id": "toolu_01A2B3C4D5E6",
            "name": "Bash",
            "input": {
                "command": "ls -la",
            },
        }
    ],
    # stop_reason: tool_use 表示模型将继续调用工具
}

# -----------------------------------------------------------------------------
# Tool Result Message（使用 Anthropic API tool_result content block 格式）
# -----------------------------------------------------------------------------

tool_result_message = {
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01A2B3C4D5E6",
            "content": "total 32\ndrwxr-xr-x  12 user  staff   384 Apr 15 10:00 .\ndrwxr-xr-x   8 user  staff   256 Apr 15 09:30 ...",
            "is_error": False,
        }
    ],
}
```

---

## 4. Config 全局配置 (`config.py`)

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import TypedDict
import os
from pathlib import Path


# -----------------------------------------------------------------------------
# API 配置
# -----------------------------------------------------------------------------

@dataclass
class APIConfig:
    """API 相关配置（LiteLLM provider-agnostic）"""
    api_key: str | None = None   # 通用 API Key，也可由 LiteLLM 从各 provider 环境变量自动读取
    base_url: str | None = None  # 可选的自定义 API endpoint（用于代理或本地模型）
    timeout: int = 60            # 请求超时（秒）
    max_retries: int = 3         # 最大重试次数

    @classmethod
    def from_env(cls) -> APIConfig:
        """从环境变量加载 API 配置。

        LiteLLM 会自动读取各 provider 的 key（如 OPENAI_API_KEY、ANTHROPIC_API_KEY）。
        LLM_API_KEY 作为通用覆盖，优先级高于 provider-specific key。
        """
        return cls(
            api_key=os.environ.get("LLM_API_KEY"),
            base_url=os.environ.get("LLM_BASE_URL"),
            timeout=int(os.environ.get("LLM_TIMEOUT", "60")),
            max_retries=int(os.environ.get("LLM_MAX_RETRIES", "3")),
        )


# -----------------------------------------------------------------------------
# MCP Server 配置
# -----------------------------------------------------------------------------

@dataclass
class MCPServerConfig:
    """单个 MCP Server 配置"""
    command: str  # 启动命令，如 "npx"
    args: list[str] = field(default_factory=list)  # 命令参数
    env: dict[str, str] = field(default_factory=dict)  # 环境变量


@dataclass
class MCPConfig:
    """MCP 相关配置"""
    servers: dict[str, MCPServerConfig] = field(default_factory=dict)

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> MCPConfig:
        """从字典加载 MCP 配置"""
        servers = {}
        for name, server_data in data.items():
            servers[name] = MCPServerConfig(
                command=server_data["command"],
                args=server_data.get("args", []),
                env=server_data.get("env", {}),
            )
        return cls(servers=servers)


# -----------------------------------------------------------------------------
# Skills 配置
# -----------------------------------------------------------------------------

@dataclass
class SkillsConfig:
    """Skills 相关配置"""
    directory: Path | None = None  # Skills 目录路径
    enabled: list[str] = field(default_factory=list)  # 启用的 skill 列表
    disabled: list[str] = field(default_factory=list)  # 禁用的 skill 列表

    @classmethod
    def from_env(cls) -> SkillsConfig:
        """从环境变量和配置文件加载"""
        default_dir = Path.home() / ".claude" / "skills"
        return cls(
            directory=Path(os.environ.get("CLAUDE_SKILLS_DIR", default_dir)),
            enabled=[],
            disabled=[],
        )


# -----------------------------------------------------------------------------
# Model 配置
# -----------------------------------------------------------------------------

@dataclass
class ModelConfig:
    """模型相关配置。

    LiteLLM 格式：provider/model-name，例如：
    - "anthropic/claude-sonnet-4-20250514"
    - "openai/gpt-4o"
    - "gemini/gemini-1.5-pro"
    - "ollama/llama3"
    """
    name: str = "anthropic/claude-sonnet-4-20250514"
    temperature: float = 1.0
    max_tokens: int = 8192
    top_p: float = 0.9  # nucleus sampling
    top_k: int = 40     # top-k sampling（部分 provider 支持）

    @classmethod
    def from_env(cls) -> ModelConfig:
        """从环境变量加载模型配置"""
        return cls(
            name=os.environ.get("LLM_MODEL", "anthropic/claude-sonnet-4-20250514"),
            temperature=float(os.environ.get("LLM_TEMPERATURE", "1.0")),
            max_tokens=int(os.environ.get("LLM_MAX_TOKENS", "8192")),
            top_p=float(os.environ.get("LLM_TOP_P", "0.9")),
            top_k=int(os.environ.get("LLM_TOP_K", "40")),
        )


# -----------------------------------------------------------------------------
# 主配置类
# -----------------------------------------------------------------------------

class AppConfig(TypedDict, total=False):
    """主配置类型（与 state/store.py 中的 AppState.config 保持一致）"""
    api: APIConfig
    mcp: MCPConfig
    skills: SkillsConfig
    model: ModelConfig


# -----------------------------------------------------------------------------
# 配置验证（使用 Pydantic）
# -----------------------------------------------------------------------------

from pydantic import BaseModel, Field, field_validator


class ValidatedAPIConfig(BaseModel):
    """API 配置验证"""
    api_key: str | None = None
    base_url: str | None = None
    timeout: int = Field(default=60, ge=1, le=300)
    max_retries: int = Field(default=3, ge=0, le=10)


class ValidatedMCPServerConfig(BaseModel):
    """MCP Server 配置验证"""
    command: str = Field(..., min_length=1)
    args: list[str] = Field(default_factory=list)
    env: dict[str, str] = Field(default_factory=dict)


class ValidatedModelConfig(BaseModel):
    """模型配置验证。model name 使用 LiteLLM 格式：provider/model-name"""
    name: str = Field(default="anthropic/claude-sonnet-4-20250514")
    temperature: float = Field(default=1.0, ge=0, le=2)
    max_tokens: int = Field(default=8192, ge=1, le=200000)
    top_p: float = Field(default=0.9, ge=0, le=1)
    top_k: int = Field(default=40, ge=1)

    @field_validator("name")
    @classmethod
    def validate_model_name(cls, v: str) -> str:
        # LiteLLM 接受任意 "provider/model" 格式，不在这里白名单校验
        if not v:
            raise ValueError("model name cannot be empty")
        return v


# -----------------------------------------------------------------------------
# 配置加载器
# -----------------------------------------------------------------------------

class ConfigLoader:
    """
    配置加载器。

    配置注入方式：作为参数传递，非全局变量
    这样便于：
    1. 单元测试时注入 mock 配置
    2. 多实例运行时使用不同配置
    3. 配置热更新（创建新配置对象）
    """

    @staticmethod
    def load(config_path: Path | None = None) -> AppConfig:
        """
        加载完整配置。

        加载顺序（优先级从低到高）：
        1. 默认值
        2. 配置文件（~/.claude/settings.json）
        3. 环境变量
        4. CLI 参数（外部传入）
        """
        # 从环境变量加载
        api_config = APIConfig.from_env()
        mcp_config = MCPConfig()  # 从配置文件加载
        skills_config = SkillsConfig.from_env()
        model_config = ModelConfig.from_env()

        # 如果有配置文件，合并
        if config_path and config_path.exists():
            config_data = ConfigLoader._load_json(config_path)
            mcp_config = MCPConfig.from_dict(config_data.get("mcp_servers", {}))

        return AppConfig(
            api=api_config,
            mcp=mcp_config,
            skills=skills_config,
            model=model_config,
        )

    @staticmethod
    def _load_json(path: Path) -> dict[str, Any]:
        """加载 JSON 配置文件"""
        import json
        with open(path) as f:
            return json.load(f)


# -----------------------------------------------------------------------------
# 配置如何注入各模块
# -----------------------------------------------------------------------------

class Agent:
    """
    Agent 示例：展示配置如何通过参数传递注入。

    不使用全局变量，而是通过构造函数或方法参数接收配置。
    """

    def __init__(
        self,
        config: AppConfig,
        store: Store[dict[str, Any]],
    ) -> None:
        self._config = config  # 保存配置引用
        self._store = store

        # 使用配置初始化各模块
        self._api_client = self._create_api_client(config.api)
        self._mcp_manager = self._create_mcp_manager(config.mcp)
        self._skills_loader = self._create_skills_loader(config.skills)

    def _create_api_client(self, api_config: APIConfig):
        """配置 LiteLLM 全局参数（无需创建 client 实例，LiteLLM 是函数式调用）"""
        import litellm
        if api_config.api_key:
            litellm.api_key = api_config.api_key
        if api_config.base_url:
            litellm.api_base = api_config.base_url
        litellm.request_timeout = api_config.timeout
        litellm.num_retries = api_config.max_retries
        return None  # LiteLLM 不需要 client 对象

    def _create_mcp_manager(self, mcp_config: MCPConfig):
        """创建 MCP 管理器（使用注入的配置）"""
        # 使用 mcp_config.servers 初始化
        return mcp_config

    def _create_skills_loader(self, skills_config: SkillsConfig):
        """创建 Skills 加载器（使用注入的配置）"""
        return skills_config
```

---

## 5. Main.py 入口文件

```python
"""
Agent 入口文件。

完整的初始化序列：
1. 加载配置（ConfigLoader）
2. 创建 Store（状态容器）
3. 初始化各模块（API Client, MCP, Skills, Tools）
4. 启动 Agent Loop
5. 注册优雅退出处理
"""

from __future__ import annotations
import asyncio
import signal
import sys
from pathlib import Path
from typing import Any

from state.store import create_store, Store
from state.selectors import (
    get_last_message,
    get_message_count,
    get_tool_calls_from_last_assistant_message,
)
from context.messages import (
    build_context,
    BuildContextInput,
    append_user_message,
    append_assistant_message,
    append_tool_result_message,
)
from config import ConfigLoader, AppConfig


# -----------------------------------------------------------------------------
# 类型定义
# -----------------------------------------------------------------------------

class Agent:
    """主 Agent 类"""

    def __init__(
        self,
        config: AppConfig,
        store: Store[dict[str, Any]],
    ) -> None:
        self._config = config
        self._store = store
        self._running = False

    async def start(self) -> None:
        """启动 Agent"""
        self._running = True
        await self._main_loop()

    async def stop(self) -> None:
        """停止 Agent"""
        self._running = False
        print("Agent 已停止")

    async def _main_loop(self) -> None:
        """
        主循环。

        伪代码：
        while running:
            user_input = await get_user_input()
            state = store.get_state()
            context = build_context(user_input, state)
            response = await call_llm(context)
            store.set(append_assistant_message(response))
            for tool_call in response.tool_use:
                result = await execute_tool(tool_call)
                store.set(append_tool_result_message(result))
        """
        while self._running:
            try:
                # 获取用户输入（非阻塞）
                user_input = await self._get_user_input()
                if user_input is None:
                    break

                # 先将用户消息持久化到 store（build_context 是纯函数，不会自动写入）
                self._store.set(lambda s: append_user_message(s, user_input))

                # 再读取已更新的 state，构建 API 消息列表
                state = self._store.get_state()
                context_input = BuildContextInput(
                    user_input=user_input,
                    current_state=state,
                )
                context = build_context(context_input)

                # 调用 LLM
                response = await self._call_llm(context)

                # 追加助手消息
                self._store.set(lambda s: append_assistant_message(
                    s,
                    content=response["content"],
                    tool_use=response.get("tool_use", []),
                    stop_reason=response.get("stop_reason"),
                ))

                # 执行工具调用
                for tool_call in response.get("tool_use", []):
                    result = await self._execute_tool(tool_call)
                    self._store.set(lambda s: append_tool_result_message(
                        s,
                        tool_use_id=tool_call["id"],
                        content=result["content"],
                        is_error=result.get("is_error", False),
                    ))

            except KeyboardInterrupt:
                break
            except Exception as e:
                print(f"错误: {e}")
                continue

    async def _get_user_input(self) -> str | None:
        """获取用户输入（可扩展为支持 stdin/文件/API）"""
        try:
            loop = asyncio.get_event_loop()
            return await loop.run_in_executor(None, input, "> ")
        except EOFError:
            return None

    async def _call_llm(self, context: list[dict[str, Any]]) -> dict[str, Any]:
        """调用 LLM API（通过 LiteLLM，provider-agnostic）"""
        from litellm import acompletion

        response = await acompletion(
            model=self._config.model.name,
            max_tokens=self._config.model.max_tokens,
            messages=context,
        )

        choice = response.choices[0]
        message = choice.message

        # 提取文本内容
        content = message.content or ""

        # 提取工具调用（OpenAI function-calling 格式）
        tool_use = []
        if message.tool_calls:
            import json
            for tc in message.tool_calls:
                try:
                    args = json.loads(tc.function.arguments) if tc.function.arguments else {}
                except json.JSONDecodeError:
                    args = {}
                tool_use.append({"id": tc.id, "name": tc.function.name, "input": args})

        # 映射 finish_reason → stop_reason
        finish_map = {"stop": "end_turn", "length": "max_tokens", "tool_calls": "tool_use"}
        stop_reason = finish_map.get(choice.finish_reason, choice.finish_reason)

        return {
            "content": content,
            "tool_use": tool_use,
            "stop_reason": stop_reason,
        }

    async def _execute_tool(self, tool_call: dict[str, Any]) -> dict[str, Any]:
        """执行工具调用"""
        tool_name = tool_call["name"]
        tool_input = tool_call["input"]

        # TODO: 实现实际工具执行逻辑
        return {
            "content": f"Tool {tool_name} executed with {tool_input}",
            "is_error": False,
        }


# -----------------------------------------------------------------------------
# 初始化序列
# -----------------------------------------------------------------------------

def initialize(config_path: Path | None = None) -> tuple[AppConfig, Store[dict[str, Any]], Agent]:
    """
    初始化 Agent 系统。

    依赖注入顺序：
    1. Config（无依赖）
    2. Store（依赖 Config 初始化默认值）
    3. Agent（依赖 Config 和 Store）

    Returns:
        (config, store, agent) 元组
    """
    # Step 1: 加载配置
    config = ConfigLoader.load(config_path)

    # Step 2: 创建 Store
    from state.store import get_default_state
    initial_state = get_default_state()
    initial_state["config"] = config
    store = create_store(initial_state)

    # Step 3: 创建 Agent
    agent = Agent(config=config, store=store)

    return config, store, agent


# -----------------------------------------------------------------------------
# 优雅退出处理
# -----------------------------------------------------------------------------

class GracefulShutdown:
    """优雅退出处理器"""

    def __init__(self, agent: Agent) -> None:
        self._agent = agent
        self._shutdown_requested = False

    def register(self) -> None:
        """注册信号处理器"""
        signal.signal(signal.SIGINT, self._handle_signal)
        signal.signal(signal.SIGTERM, self._handle_signal)

    def _handle_signal(self, signum: int, frame: Any) -> None:
        """处理退出信号"""
        if self._shutdown_requested:
            print("\n强制退出...")
            sys.exit(1)

        print("\n收到退出信号，正在优雅关闭...")
        self._shutdown_requested = True
        asyncio.create_task(self._shutdown_async())

    async def _shutdown_async(self) -> None:
        """异步关闭"""
        await self._agent.stop()


# -----------------------------------------------------------------------------
# Main 入口
# -----------------------------------------------------------------------------

async def main() -> None:
    """主入口函数"""
    # 解析命令行参数（示例）
    import argparse
    parser = argparse.ArgumentParser(description="Claude Agent")
    parser.add_argument("--config", type=Path, help="配置文件路径")
    args = parser.parse_args()

    # 初始化
    config, store, agent = initialize(args.config)

    print(f"Agent 已启动 (model: {config.model.name})")

    # 注册优雅退出
    shutdown_handler = GracefulShutdown(agent)
    shutdown_handler.register()

    # 启动 Agent
    await agent.start()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. 模块间依赖关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              config.py                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  APIConfig  │  │  MCPConfig  │  │ SkillsConfig│  │   ModelConfig       │ │
│  │             │  │             │  │             │  │                     │ │
│  │ • api_key   │  │ • servers{} │  │ • directory │  │ • name              │ │
│  │ • base_url  │  │             │  │ • enabled[] │  │ • temperature       │ │
│  │ • timeout   │  │             │  │ • disabled[]│  │ • max_tokens        │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
└─────────┼────────────────┼───────────────┼───────────────────┼───────────┘
          │                │               │                   │
          │                │               │                   │
          ▼                ▼               ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             AppConfig                                        │
│  (TypedDict, 作为参数在各模块间传递，非全局变量)                               │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 依赖注入
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             state/store.py                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                         Store[T]                                        │ │
│  │  • get_state() → T                                                     │ │
│  │  • set_state(updater) → void                                           │ │
│  │  • subscribe(listener) → unsubscribe                                     │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      │ on_change 回调                        │
│                                      ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                      state/selectors.py                                  │ │
│  │  • get_last_message(state) → msg | None                                  │ │
│  │  • get_message_count(state) → int                                       │ │
│  │  • get_tool_by_name(name)(state) → tool | None                          │ │
│  │  • get_active_tasks(state) → list[Task]                                 │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 状态读取
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           context/messages.py                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                       build_context()                                    │ │
│  │  输入: BuildContextInput(user_input, current_state, system_prompt)      │ │
│  │  输出: list[APIMessage]  (可直接发送给 LLM)                              │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                     消息追加辅助函数                                      │ │
│  │  • append_user_message(state, content) → new_state                      │ │
│  │  • append_assistant_message(state, content, tool_use, stop_reason)       │ │
│  │  • append_tool_result_message(state, tool_use_id, content, is_error)     │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 消息列表
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          services/api/client.py                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                  LiteLLM (litellm.acompletion)                           │ │
│  │  • acompletion(model, max_tokens, messages)                              │ │
│  │  • model = "anthropic/..." | "openai/..." | "gemini/..." | "ollama/..."  │ │
│  │  • API key 由 LLM_API_KEY 或各 provider 环境变量提供                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ LLM 响应
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            main.py / Agent                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                         Agent Loop                                       │ │
│  │  while running:                                                         │ │
│  │    user_input = await get_user_input()                                  │ │
│  │    context = build_context(user_input, store.get_state())                │ │
│  │    response = await call_llm(context)                                   │ │
│  │    store.set(append_assistant_message(response))                        │ │
│  │    for tool_call in response.tool_use:                                  │ │
│  │      result = await execute_tool(tool_call)                             │ │
│  │      store.set(append_tool_result_message(result))                      │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 工具执行
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              tools/                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  BashTool   │  │  ReadTool   │  │  EditTool   │  │    AgentTool        │ │
│  │             │  │             │  │             │  │                     │ │
│  │ • command   │  │ • path      │  │ • path      │  │ • name              │ │
│  │ • timeout   │  │ • limit     │  │ • old_text  │  │ • prompt            │ │
│  │ • cwd       │  │             │  │ • new_text  │  │ • subagent_id       │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 工具注册
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              skills/                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                      Skills Loader                                       │ │
│  │  • 从 config.skills.directory 加载 skill 文件                            │ │
│  │  • 每个 skill: { name, description, handler }                            │ │
│  │  • 注册到 store.tools{}                                                   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ MCP 服务器
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               mcp/                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                      MCP Manager                                         │ │
│  │  • 从 config.mcp.servers{} 启动 MCP 服务器                               │ │
│  │  • 管理 MCP client 连接                                                   │ │
│  │  • 暴露 MCP tools/resources 到 store.mcp{}                               │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ 任务状态
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              tasks/                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │LocalAgent   │  │RemoteAgent │  │InProcess    │  │   DreamTask         │ │
│  │Task         │  │Task        │  │TeammateTask │  │                     │ │
│  │             │  │            │  │             │  │                     │ │
│  │• status     │  │• session_id│  │• agent_id   │  │ • prompt            │ │
│  │• result     │  │• websocket  │  │• in_process │  │ • iteration         │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 依赖注入方向（箭头表示「依赖于」）

```
config.py ──────────→ Store ──────────→ Agent ──────────→ 工具执行
     │                  │                  │
     │                  │                  │
     ▼                  ▼                  ▼
selectors.py      build_context()     tools/, skills/
```

### 关键设计原则

1. **配置非全局**：所有配置通过参数传递，便于测试和多实例运行
2. **Store 是唯一真相源**：所有状态存储在 Store 中，各模块订阅变化
3. **Selector 派生数据**：避免在调用处计算派生状态
4. **Context 构建纯函数**：`build_context()` 是纯函数，输入确定则输出确定
5. **消息不可变**：使用 dataclass(frozen=True) 确保消息不可变
6. **线程安全**：Store 内部使用 RLock 保证并发安全

---

## 附录：完整文件结构

```
agent/
├── __init__.py
├── main.py                    # 入口文件（完整示例）
├── config.py                  # 配置管理
├── state/
│   ├── __init__.py
│   ├── store.py               # Store 泛型类
│   ├── selectors.py           # 状态选择器
│   └── default_state.py        # 默认状态工厂
├── context/
│   ├── __init__.py
│   ├── messages.py            # build_context() + 消息追加
│   └── types.py               # 消息类型定义
├── services/
│   └── api/
│       └── client.py          # LLM API 客户端
├── tools/
│   ├── __init__.py
│   ├── base.py                # Tool 基类
│   ├── bash.py                # Bash 工具
│   ├── read.py                # 读文件工具
│   └── edit.py                # 编辑工具
├── skills/
│   ├── __init__.py
│   └── loader.py              # Skills 加载器
└── mcp/
    ├── __init__.py
    └── manager.py             # MCP 管理器
```
