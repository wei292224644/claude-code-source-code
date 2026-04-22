# Agent Loop 模块技术文档

> 参考实现：TypeScript 版位于 `src/query.ts` 和 `src/QueryEngine.ts`
> 本文档为 Python 版本，LLM 调用层使用 **LiteLLM**（provider-agnostic）

---

## 1. 模块职责概述

### 1.1 agent/loop.py 核心职责

`loop.py` 是 Agent 的主循环引擎，负责：

1. **维护消息状态**：管理对话历史 `messages[]`，支持多轮交互
2. **while-true 循环控制**：持续调用 LLM 直到达到终止条件
3. **tool_use 调度**：从 AssistantMessage 提取 tool_use 块，路由到对应 Tool handler
4. **stop_reason 处理**：判断是 `end_turn`（正常结束）、`max_tokens`（输出超限）还是 `stop_sequence`
5. **错误恢复**：处理 API 错误、超时、token 限制等异常情况

### 1.2 agent/query_engine.py 核心职责

`query_engine.py` 是对 LiteLLM 的封装层，负责：

1. **Stream 管理**：将 SDK 的 `AsyncGenerator` 流式响应转换为 Python async generator 模式
2. **Token 计算与上报**：从流式事件中提取 `usage`（input_tokens、output_tokens、cache 统计）
3. **stop_signal 处理**：捕获 SDK 的中断信号（用户取消、外部中止）
4. **Session 生命周期**：管理多轮对话的 session 状态持久化
5. **Budget 控制**：支持 `maxTurns`、`maxBudgetUsd`、`taskBudget` 等约束

### 1.3 两者协作关系

```
loop.py (while-true 循环)
    │
    ├─── 调用 ──→ query_engine.py.submit_message()
    │                    │
    │                    ├─── 调用 ──→ SDK client.messages.create()
    │                    │                    │
    │                    │                    └─── 返回 AsyncGenerator[StreamEvent]
    │                    │
    │                    └─── yield ←── StreamEvent (增量消息块)
    │
    ├── 提取 tool_use 块
    │
    ├─── 调用 ──→ Tool.handler() → ToolResult
    │
    └─── 构造 HumanMessage → 添加到 messages[] → 继续循环
```

---

## 2. Message 类型系统 (agent/types.py)

### 2.1 Pydantic 模型定义

```python
# agent/types.py
from pydantic import BaseModel, Field, field_validator
from typing import Annotated, Literal, Any
from datetime import datetime


class HumanMessage(BaseModel):
    """用户输入消息"""
    role: Literal["user"] = "user"
    content: str | list[dict[str, Any]]
    timestamp: datetime = Field(default_factory=datetime.now)
    uuid: str | None = None
    is_meta: bool = False  # 元消息（如 max_tokens 恢复消息）
    tool_use_result: str | None = None  # 标记是否为 tool 结果

    @field_validator("content")
    @classmethod
    def content_must_not_be_empty(cls, v):
        if isinstance(v, str) and not v.strip():
            raise ValueError("content cannot be empty string")
        return v


class ToolUseBlock(BaseModel):
    """LLM 生成的 tool_use 块"""
    type: Literal["tool_use"] = "tool_use"
    id: str = Field(description="工具调用唯一 ID，用于匹配结果")
    name: str = Field(description="工具名称")
    input: dict[str, Any] = Field(description="工具输入参数")

    # 验证：input 必须是字典
    @field_validator("input")
    @classmethod
    def input_must_be_dict(cls, v):
        if not isinstance(v, dict):
            raise ValueError(f"tool_use input must be dict, got {type(v)}")
        return v


class AssistantMessage(BaseModel):
    """LLM 响应消息"""
    role: Literal["assistant"] = "assistant"
    content: str | list[ToolUseBlock | dict[str, Any]] = Field(
        description="响应内容：纯文本 或 content block 列表"
    )
    timestamp: datetime = Field(default_factory=datetime.now)
    uuid: str | None = None
    stop_reason: Literal["end_turn", "max_tokens", "stop_sequence", "tool_use"] | None = None
    is_api_error_message: bool = False  # API 错误标记

    def get_tool_uses(self) -> list[ToolUseBlock]:
        """提取所有 tool_use 块"""
        if isinstance(self.content, str):
            return []
        return [block for block in self.content if isinstance(block, ToolUseBlock)]


class ToolResultMessage(BaseModel):
    """工具执行结果（作为 HumanMessage 发送给 LLM）"""
    role: Literal["user"] = "user"
    content: list[dict[str, Any]] = Field(
        description="tool_result 块列表，结构同 Anthropic SDK"
    )
    tool_use_id: str = Field(description="关联的 tool_use ID")
    timestamp: datetime = Field(default_factory=datetime.now)
    uuid: str | None = None
    is_error: bool = False


class SystemMessage(BaseModel):
    """系统消息（如 API 重试警告、compaction 边界）"""
    type: Literal["system"] = "system"
    subtype: str  # 如 "api_retry", "compact_boundary"
    content: str | None = None
    timestamp: datetime = Field(default_factory=datetime.now)
    uuid: str | None = None
    # API 重试信息
    retry_attempt: int | None = None
    max_retries: int | None = None
    retry_delay_ms: int | None = None


class StreamEvent(BaseModel):
    """流式事件（SDK 增量输出）"""
    type: Literal["stream_event"] = "stream_event"
    event: dict[str, Any] = Field(
        description="原始 SDK event，常见类型：message_start, content_block_start, "
                    "content_block_delta, content_block_stop, message_delta, message_stop"
    )
    session_id: str | None = None
    parent_tool_use_id: str | None = None
    uuid: str | None = None
```

### 2.2 字段验证规则

| 字段 | 验证规则 | 错误处理 |
|------|----------|----------|
| `HumanMessage.content` | 非空字符串或非空列表 | `ValueError` |
| `ToolUseBlock.input` | 必须是 `dict` 类型 | `ValueError` |
| `ToolUseBlock.id` | 非空字符串 | SDK 保证 |
| `AssistantMessage.stop_reason` | 枚举值之一或 None | SDK 保证 |

### 2.3 示例代码

```python
# 创建用户消息
user_msg = HumanMessage(
    content="帮我写一个快速排序算法",
    uuid="msg-001"
)

# 创建 AssistantMessage（带 tool_use）
assistant_msg = AssistantMessage(
    content=[
        {"type": "text", "text": "我来帮你写。"},
        {
            "type": "tool_use",
            "id": "toolu_01",
            "name": "WebFetch",
            "input": {"url": "https://example.com"}
        }
    ],
    stop_reason="tool_use",
    uuid="msg-002"
)

# 提取 tool_use
tool_uses = assistant_msg.get_tool_uses()
print(tool_uses[0].name)  # "WebFetch"
print(tool_uses[0].input)  # {"url": "https://example.com"}

# 创建 ToolResultMessage
tool_result = ToolResultMessage(
    content=[{
        "type": "tool_result",
        "tool_use_id": "toolu_01",
        "content": "网页内容...",
        "is_error": False
    }],
    tool_use_id="toolu_01",
    is_error=False
)
```

---

## 3. 主循环流程 (agent/loop.py)

### 3.1 run_agent_loop() 完整实现

```python
# agent/loop.py
import asyncio
import dataclasses
from typing import AsyncGenerator, Annotated
from dataclasses import dataclass, field
import uuid

from agent.types import (
    HumanMessage,
    AssistantMessage,
    ToolResultMessage,
    SystemMessage,
    StreamEvent,
    ToolUseBlock,
)
from agent.query_engine import QueryEngine, QueryEngineConfig
from agent.tool_execution import execute_tool_safe, _error_result
from tools.registry import ToolRegistry, get_tool_registry
from tools.tool import Tool, ToolUseContext, ToolResult


@dataclass
class LoopConfig:
    """循环配置项"""
    max_turns: int | None = None           # 最大轮次限制
    max_budget_usd: float | None = None    # 最大 USD 预算
    task_budget: dict | None = None        # task_budget（API 内部预算）
    tools: list[Tool] = field(default_factory=list)
    system_prompt: str | None = None


@dataclass
class LoopState:
    """循环状态（每轮迭代时重新解构）"""
    messages: list[HumanMessage | AssistantMessage | ToolResultMessage | SystemMessage]
    tool_use_context: ToolUseContext
    turn_count: int = 1
    total_usage: dict = field(default_factory=lambda: {
        "input_tokens": 0,
        "output_tokens": 0,
        "cache_creation_input_tokens": 0,
        "cache_read_input_tokens": 0,
    })


async def run_agent_loop(
    initial_messages: list[HumanMessage | AssistantMessage],
    config: LoopConfig,
) -> AsyncGenerator[
    AssistantMessage | ToolResultMessage | SystemMessage | StreamEvent,
    AssistantMessage | None  # 最终返回值
]:
    """
    Agent 主循环。

    Args:
        initial_messages: 初始消息列表（通常只有一条 HumanMessage）
        config: 循环配置

    Yields:
        - AssistantMessage: LLM 增量输出
        - ToolResultMessage: 工具执行结果
        - SystemMessage: 系统消息（API 重试、边界等）
        - StreamEvent: SDK 流式事件（可选，取决于 include_partial_messages）

    Returns:
        最终一条 AssistantMessage 或 None
    """
    # 初始化状态
    state = LoopState(
        messages=list(initial_messages),
        tool_use_context=_build_tool_use_context(config),
    )

    # 构建 QueryEngine 配置
    qe_config = QueryEngineConfig(
        tools=config.tools,
        system_prompt=config.system_prompt or _get_default_system_prompt(),
        max_turns=config.max_turns,
        max_budget_usd=config.max_budget_usd,
        task_budget=config.task_budget,
        tool_use_context=state.tool_use_context,
    )

    engine = QueryEngine(qe_config)

    while True:
        # === 每轮迭代开始：解构状态 ===
        tool_use_context = state.tool_use_context
        messages = state.messages
        turn_count = state.turn_count

        # === 构建发送给 LLM 的消息列表 ===
        # 过滤：移除 tombstone（墓碑消息，控制信号）
        # 截断：compaction 后的消息可能需要裁剪
        messages_for_query = _prepare_messages_for_query(messages)

        # === 调用 LLM（通过 QueryEngine）===
        current_usage: dict = {
            "input_tokens": 0,
            "output_tokens": 0,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 0,
        }

        try:
            async for event in engine.submit_message(messages_for_query):
                # ---- 流式事件处理 ----
                if event.type == "stream_event":
                    # 提取 token 使用量
                    _accumulate_usage(current_usage, event.event)
                    yield StreamEvent(
                        type="stream_event",
                        event=event.event,
                    )
                    continue

                # ---- AssistantMessage 处理 ----
                if event.type == "assistant":
                    # 先持久化到 state（不可变更新），再 yield——避免 state.messages[-1] 读到旧消息
                    state = dataclasses.replace(state, messages=[*state.messages, event])
                    yield event

                    # 检查是否需要 follow-up（tool_use 块存在）
                    tool_use_blocks = event.get_tool_uses()
                    if tool_use_blocks:
                        # 有 tool_use：执行工具，跳过常规终止检查
                        break
                    else:
                        # 无 tool_use：检查 stop_reason
                        if event.stop_reason == "end_turn":
                            # 正常结束：执行 stop hooks，返回
                            return event
                        elif event.stop_reason == "max_tokens":
                            # max_tokens：注入恢复消息，继续循环
                            recovery_msg = _create_max_tokens_recovery_message()
                            state = dataclasses.replace(
                                state,
                                messages=[*state.messages, recovery_msg],
                                turn_count=state.turn_count + 1,
                            )
                            continue
                        elif event.stop_reason == "stop_sequence":
                            # stop_sequence：按 stop_sequence 处理
                            return event

                # ---- ToolResultMessage 处理 ----
                if event.type == "user" and hasattr(event, "tool_use_id"):
                    yield event
                    continue

                # ---- SystemMessage 处理 ----
                if event.type == "system":
                    yield event
                    # API 重试：继续等待
                    if event.subtype == "api_retry":
                        continue

        except asyncio.CancelledError:
            # 用户中断（Ctrl+C）
            raise

        except Exception as e:
            # API 调用失败
            error_msg = _handle_api_error(e, state.messages)
            state = dataclasses.replace(
                state,
                messages=[*state.messages, error_msg],
                turn_count=state.turn_count + 1,
            )
            continue

        # === 执行 tool_use 块 ===
        # AssistantMessage 已在上方持久化，state.messages[-1] 此时一定是它
        assistant_msg = state.messages[-1] if state.messages else None
        if not isinstance(assistant_msg, AssistantMessage):
            continue

        tool_use_blocks = assistant_msg.get_tool_uses()
        registry = get_tool_registry()

        # 并发执行所有工具（asyncio.gather 实现真正并行，不再串行等待）
        async def _run_one(block: ToolUseBlock) -> ToolResultMessage:
            tool = registry.get(block.name)
            if tool is None:
                return _error_result(block.id, f"Tool '{block.name}' not found", is_recoverable=False)
            return await execute_tool_safe(tool_block=block, tool=tool, context=tool_use_context)

        tool_results: list[ToolResultMessage] = list(
            await asyncio.gather(*[_run_one(b) for b in tool_use_blocks])
        )

        # 不可变更新：一次性将所有结果追加到消息历史
        state = dataclasses.replace(state, messages=[*state.messages, *tool_results])
        for result in tool_results:
            yield result

        # === 检查中止条件 ===
        # max_turns 限制
        if config.max_turns and state.turn_count >= config.max_turns:
            yield SystemMessage(
                subtype="max_turns_reached",
                content=f"Reached maximum number of turns ({config.max_turns})",
            )
            return None

        # max_budget_usd 限制
        if config.max_budget_usd and _get_total_cost(state.total_usage) >= config.max_budget_usd:
            yield SystemMessage(
                subtype="max_budget_reached",
                content=f"Reached maximum budget (${config.max_budget_usd})",
            )
            return None

        # === 轮次递增，继续循环（不可变更新）===
        state = dataclasses.replace(
            state,
            turn_count=state.turn_count + 1,
            total_usage=_merge_usage(state.total_usage, current_usage),
        )


def _prepare_messages_for_query(
    messages: list[HumanMessage | AssistantMessage | ToolResultMessage | SystemMessage],
) -> list[HumanMessage | AssistantMessage | ToolResultMessage]:
    """
    过滤并准备发送给 LLM 的消息列表。

    过滤规则：
    - 移除 SystemMessage（控制信号，不发送给 LLM）
    - 保留 HumanMessage、AssistantMessage、ToolResultMessage
    """
    return [msg for msg in messages if not isinstance(msg, SystemMessage)]


def _build_tool_use_context(config: LoopConfig) -> ToolUseContext:
    """构建 ToolUseContext"""
    return ToolUseContext(
        options={
            "tools": config.tools,
            "main_loop_model": "claude-sonnet-4-20250514",
            "thinking_config": {"type": "disabled"},
            "is_non_interactive_session": True,
            "max_budget_usd": config.max_budget_usd,
        },
        abort_controller=asyncio.Event(),
        # ... 其他必要字段
    )


def _create_max_tokens_recovery_message() -> HumanMessage:
    """创建 max_tokens 恢复消息"""
    return HumanMessage(
        content=(
            "Output token limit hit. Resume directly — no apology, no recap. "
            "Pick up mid-thought if that is where the cut happened. "
            "Break remaining work into smaller pieces."
        ),
        is_meta=True,
    )


def _handle_api_error(
    error: Exception,
    messages: list[HumanMessage | AssistantMessage],
) -> HumanMessage:
    """处理 API 错误"""
    return HumanMessage(
        content=f"API Error: {str(error)}",
        is_meta=True,
    )
```

### 3.2 while-true 循环退出条件

| 退出路径 | 条件 | 处理 |
|----------|------|------|
| `return assistant_msg` | `stop_reason == "end_turn"` 且无 tool_use | 正常完成 |
| `return assistant_msg` | `stop_reason == "stop_sequence"` | 遇到停止序列 |
| `return None` | `max_turns` 达到限制 | 轮次耗尽 |
| `return None` | `max_budget_usd` 达到限制 | 预算耗尽 |
| `raise CancelledError` | 用户中断（Ctrl+C） | 外部取消 |
| `continue` | `stop_reason == "max_tokens"` | 注入恢复消息，继续 |

### 3.3 stop_reason 处理详解

```python
# stop_reason 类型定义
StopReason = Literal["end_turn", "max_tokens", "stop_sequence", "tool_use"]

# stop_reason 处理逻辑
def _handle_stop_reason(
    stop_reason: StopReason | None,
    assistant_msg: AssistantMessage,
) -> None | AssistantMessage:
    """返回最终消息或 None（表示继续循环）"""

    if stop_reason == "end_turn":
        # 正常结束：检查是否有 tool_use
        if assistant_msg.get_tool_uses():
            # 有 tool_use：需要执行工具，不结束
            return None
        # 无 tool_use：正常结束
        return assistant_msg

    if stop_reason == "max_tokens":
        # 输出超限：注入恢复消息，继续循环
        return None

    if stop_reason == "stop_sequence":
        # 遇到停止序列：立即结束
        return assistant_msg

    # stop_reason 为 None：可能是流式传输中尚未确定
    # 检查内容是否有 tool_use
    if assistant_msg.get_tool_uses():
        return None
    return assistant_msg
```

---

## 4. QueryEngine 封装 (agent/query_engine.py)

### 4.1 封装 LiteLLM 客户端

LiteLLM 是 provider-agnostic 的 LLM 调用库，提供统一的 OpenAI 兼容接口。通过设置 `model` 参数前缀（如 `"anthropic/claude-sonnet-4-20250514"`、`"openai/gpt-4o"`），可以透明地切换底层提供商，无需改动调用代码。

```python
# agent/query_engine.py
import asyncio
import os
import json
from typing import AsyncGenerator, Any
from dataclasses import dataclass, field
from datetime import datetime
import uuid

import litellm
from litellm import acompletion


@dataclass
class QueryEngineConfig:
    """QueryEngine 配置项"""
    tools: list[Any] = field(default_factory=list)
    system_prompt: str = ""
    max_turns: int | None = None
    max_budget_usd: float | None = None
    task_budget: dict | None = None
    tool_use_context: dict = field(default_factory=dict)
    # 回调
    on_usage: callable | None = None
    on_status: callable | None = None


class QueryEngine:
    """
    QueryEngine 封装 LiteLLM，管理流式响应和 session 状态。

    职责：
    1. 通过 LiteLLM 调用任意 LLM 提供商（Anthropic、OpenAI、Gemini 等）
    2. 将流式响应转换为内部消息格式
    3. Token 计算与上报
    4. stop_signal 处理（中断信号）

    模型选择：
    - Anthropic: "anthropic/claude-sonnet-4-20250514"
    - OpenAI:    "openai/gpt-4o"
    - Gemini:    "gemini/gemini-1.5-pro"
    - 本地 Ollama: "ollama/llama3"
    """

    def __init__(self, config: QueryEngineConfig):
        self.config = config
        self._session_id: str = str(uuid.uuid4())
        self._total_usage: dict = {
            "input_tokens": 0,
            "output_tokens": 0,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 0,
        }
        self._abort_controller: asyncio.Event | None = None

    async def submit_message(
        self,
        messages: list[dict[str, Any]],
    ) -> AsyncGenerator[Any, None]:
        """
        发送消息到 LLM，返回流式响应。

        Args:
            messages: 消息列表，OpenAI 格式（LiteLLM 兼容）

        Yields:
            - StreamEvent: 流式事件
            - AssistantMessage: 组装后的完整 assistant 消息
            - SystemMessage: 系统消息
        """
        # 转换消息格式
        api_messages = [_to_api_message(m) for m in messages]

        # 添加 system prompt
        if self.config.system_prompt:
            api_messages.insert(0, {"role": "system", "content": self.config.system_prompt})

        # 构建工具列表（OpenAI function-calling 格式，LiteLLM 自动转换）
        tools_param = [_to_litellm_tool(t) for t in self.config.tools] or None

        # 创建中止控制器
        abort_event = asyncio.Event()
        self._abort_controller = abort_event

        model = self.config.tool_use_context.get(
            "main_loop_model", "anthropic/claude-sonnet-4-20250514"
        )

        # === 流式请求 ===
        accumulated_text = ""
        accumulated_tool_calls: list[dict] = []  # {id, name, arguments_str}
        current_usage: dict = {
            "input_tokens": 0,
            "output_tokens": 0,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 0,
        }
        stop_reason: str | None = None

        try:
            response = await acompletion(
                model=model,
                messages=api_messages,
                tools=tools_param,
                max_tokens=8192,
                stream=True,
            )

            async for chunk in response:
                # LiteLLM streaming chunk 结构（OpenAI 兼容）
                delta = chunk.choices[0].delta if chunk.choices else None

                if delta is None:
                    continue

                # ---- 文本增量 ----
                if delta.content:
                    accumulated_text += delta.content
                    yield StreamEvent(
                        type="stream_event",
                        event={"type": "text_delta", "text": delta.content},
                    )

                # ---- 工具调用增量 ----
                if delta.tool_calls:
                    for tc_delta in delta.tool_calls:
                        idx = tc_delta.index
                        # 扩展列表至所需长度
                        while len(accumulated_tool_calls) <= idx:
                            accumulated_tool_calls.append(
                                {"id": "", "name": "", "arguments_str": ""}
                            )
                        if tc_delta.id:
                            accumulated_tool_calls[idx]["id"] = tc_delta.id
                        if tc_delta.function and tc_delta.function.name:
                            accumulated_tool_calls[idx]["name"] = tc_delta.function.name
                        if tc_delta.function and tc_delta.function.arguments:
                            accumulated_tool_calls[idx]["arguments_str"] += (
                                tc_delta.function.arguments
                            )

                # ---- stop_reason ----
                finish_reason = chunk.choices[0].finish_reason
                if finish_reason:
                    # 统一映射到内部 stop_reason 格式
                    stop_reason = _map_finish_reason(finish_reason)

                # ---- usage（最后一个 chunk 携带）----
                if hasattr(chunk, "usage") and chunk.usage:
                    current_usage["input_tokens"] = getattr(chunk.usage, "prompt_tokens", 0)
                    current_usage["output_tokens"] = getattr(chunk.usage, "completion_tokens", 0)

            # === 流结束：组装完整 AssistantMessage ===
            content: list[dict] = []

            if accumulated_text:
                content.append({"type": "text", "text": accumulated_text})

            for tc in accumulated_tool_calls:
                try:
                    parsed_input = json.loads(tc["arguments_str"]) if tc["arguments_str"] else {}
                except json.JSONDecodeError:
                    parsed_input = {}
                content.append({
                    "type": "tool_use",
                    "id": tc["id"],
                    "name": tc["name"],
                    "input": parsed_input,
                })

            # 更新总 usage
            self._total_usage = _merge_usage(self._total_usage, current_usage)
            if self.config.on_usage:
                self.config.on_usage(current_usage)

            assistant_msg = _build_assistant_message(
                content=content,
                stop_reason=stop_reason,
            )
            yield assistant_msg

        except asyncio.CancelledError:
            yield _build_system_message(
                subtype="api_retry",
                content="Request cancelled",
            )
            raise

        except Exception as e:
            error_type = _classify_api_error(e)

            if error_type == "rate_limit":
                yield _build_system_message(
                    subtype="api_retry",
                    retry_attempt=1,
                    max_retries=3,
                    retry_delay_ms=5000,
                    content=str(e),
                )
            elif error_type == "prompt_too_long":
                yield _build_system_message(
                    subtype="api_retry",
                    content="Prompt too long",
                )
            else:
                yield _build_system_message(
                    subtype="api_error",
                    content=str(e),
                )

    def interrupt(self) -> None:
        """发送中断信号"""
        if self._abort_controller:
            self._abort_controller.set()

    def get_total_usage(self) -> dict:
        """获取累计 usage"""
        return self._total_usage.copy()

    def get_session_id(self) -> str:
        """获取 session ID"""
        return self._session_id


def _map_finish_reason(finish_reason: str) -> str:
    """将 OpenAI finish_reason 映射到内部 stop_reason"""
    mapping = {
        "stop": "end_turn",
        "length": "max_tokens",
        "tool_calls": "tool_use",
        "content_filter": "stop_sequence",
    }
    return mapping.get(finish_reason, finish_reason)


def _to_api_message(msg: Any) -> dict:
    """将内部消息格式转换为 OpenAI / LiteLLM 格式"""
    if hasattr(msg, "role"):
        return {
            "role": msg.role,
            "content": msg.content,
        }
    return msg


def _to_litellm_tool(tool: Any) -> dict:
    """将内部工具定义转换为 OpenAI function-calling 格式（LiteLLM 兼容）"""
    return {
        "type": "function",
        "function": {
            "name": getattr(tool, "name", ""),
            "description": getattr(tool, "description", ""),
            "parameters": getattr(tool, "input_schema", {}),
        },
    }


def _build_assistant_message(
    content: list[dict],
    stop_reason: str | None,
) -> AssistantMessage:
    """构建 AssistantMessage"""
    return AssistantMessage(
        role="assistant",
        content=content,
        stop_reason=stop_reason,
        timestamp=datetime.now(),
        uuid=str(uuid.uuid4()),
    )


def _build_system_message(
    subtype: str,
    content: str | None = None,
    **kwargs,
) -> SystemMessage:
    """构建 SystemMessage"""
    return SystemMessage(
        type="system",
        subtype=subtype,
        content=content,
        timestamp=datetime.now(),
        uuid=str(uuid.uuid4()),
        **kwargs,
    )


def _classify_api_error(e: Exception) -> str:
    """分类 API 错误类型"""
    error_str = str(e).lower()
    if "rate_limit" in error_str or "429" in error_str:
        return "rate_limit"
    if "prompt_too_long" in error_str or "413" in error_str:
        return "prompt_too_long"
    return "api_error"


def _merge_usage(a: dict, b: dict) -> dict:
    """合并两个 usage 字典"""
    return {
        "input_tokens": a.get("input_tokens", 0) + b.get("input_tokens", 0),
        "output_tokens": a.get("output_tokens", 0) + b.get("output_tokens", 0),
        "cache_creation_input_tokens": a.get("cache_creation_input_tokens", 0)
                                    + b.get("cache_creation_input_tokens", 0),
        "cache_read_input_tokens": a.get("cache_read_input_tokens", 0)
                                 + b.get("cache_read_input_tokens", 0),
    }
```

### 4.2 Token 计算与上报

LiteLLM 在流结束的最后一个 chunk 中携带 `usage`（`prompt_tokens` / `completion_tokens`），在 `submit_message` 内直接读取。`_merge_usage`（定义在 4.1 节末尾）负责跨轮次累积 token 统计。

### 4.3 stop_signal（中断信号）处理

```python
async def submit_message_with_signal(
    self,
    messages: list[dict],
    signal: asyncio.Event,
) -> AsyncGenerator[Any, None]:
    """支持外部信号中断的 submit_message"""
    abort_task = None

    async def wait_for_signal():
        await signal.wait()
        # 强制重新创建 client 以中断当前请求
        self._client = None
        raise asyncio.CancelledError("User interrupted")

    try:
        # 启动信号监听任务
        abort_task = asyncio.create_task(wait_for_signal())

        # 主循环
        async for msg in self.submit_message(messages):
            yield msg

    finally:
        if abort_task:
            abort_task.cancel()
            try:
                await abort_task
            except asyncio.CancelledError:
                pass
```

---

## 5. Tool Use 处理流程

### 5.1 从 AssistantMessage 提取 tool_use 块

```python
# agent/tool_handling.py
from typing import Any
from agent.types import AssistantMessage, ToolUseBlock


def extract_tool_uses(
    assistant_msg: AssistantMessage,
) -> list[ToolUseBlock]:
    """从 AssistantMessage 提取所有 tool_use 块"""
    if isinstance(assistant_msg.content, str):
        return []

    result = []
    for block in assistant_msg.content:
        if isinstance(block, dict) and block.get("type") == "tool_use":
            result.append(ToolUseBlock(
                id=block["id"],
                name=block["name"],
                input=block["input"],
            ))
        elif hasattr(block, "type") and block.type == "tool_use":
            result.append(block)
    return result


def tool_use_dict_to_block(tool_use_dict: dict) -> ToolUseBlock:
    """将字典转换为 ToolUseBlock"""
    return ToolUseBlock(
        id=tool_use_dict["id"],
        name=tool_use_dict["name"],
        input=tool_use_dict["input"],
    )
```

### 5.2 根据 name 查找 Tool

```python
# tools/registry.py
from typing import Any
from dataclasses import dataclass, field


@dataclass
class Tool:
    """Tool 定义"""
    name: str
    aliases: list[str] = field(default_factory=list)
    description: str = ""
    input_schema: type | None = None
    handler: Any = None  # async callable

    def matches(self, name: str) -> bool:
        """检查名称是否匹配（支持别名）"""
        return self.name == name or name in self.aliases


class ToolRegistry:
    """全局 Tool 注册表"""

    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        """注册工具"""
        self._tools[tool.name] = tool
        for alias in tool.aliases:
            self._tools[alias] = tool

    def get(self, name: str) -> Tool | None:
        """根据名称查找工具（支持别名）"""
        return self._tools.get(name)

    def list_all(self) -> list[Tool]:
        """列出所有已注册工具"""
        return list(set(self._tools.values()))


# 全局注册表实例
_global_registry: ToolRegistry | None = None


def get_tool_registry() -> ToolRegistry:
    """获取全局工具注册表"""
    global _global_registry
    if _global_registry is None:
        _global_registry = ToolRegistry()
    return _global_registry


def find_tool(name: str) -> Tool | None:
    """根据名称查找 Tool（支持别名）"""
    return get_tool_registry().get(name)
```

### 5.3 构造 ToolUseContext 并传递给 handler

```python
# agent/tool_execution.py
from typing import Any, Callable
from dataclasses import dataclass, field


@dataclass
class ToolUseContext:
    """工具执行上下文"""
    options: dict[str, Any]  # 工具选项（tools 列表、model 等）
    abort_controller: Any    # asyncio.Event，用于中断
    get_app_state: Callable
    set_app_state: Callable
    messages: list[Any] = field(default_factory=list)
    query_tracking: dict | None = None
    # 以下为可选字段
    read_file_state: dict | None = None
    tool_decisions: dict | None = None
    discovered_skill_names: set[str] | None = None
    agent_id: str | None = None
    agent_type: str | None = None


class ToolExecutionError(Exception):
    """工具执行错误"""
    def __init__(
        self,
        message: str,
        tool_name: str = "",
        tool_input: dict | None = None,
        is_recoverable: bool = True,
    ):
        super().__init__(message)
        self.tool_name = tool_name
        self.tool_input = tool_input or {}
        self.is_recoverable = is_recoverable


class ValidationError(Exception):
    """输入验证错误"""
    pass


async def execute_tool(
    tool_block: ToolUseBlock,
    tool: Tool,
    context: ToolUseContext,
    assistant_message: AssistantMessage,
) -> ToolResultMessage:
    """
    执行单个工具调用。

    Args:
        tool_block: LLM 生成的 tool_use 块
        tool: 注册表中找到的 Tool
        context: 执行上下文
        assistant_message: 关联的 AssistantMessage（用于 canUseTool 钩子）

    Returns:
        ToolResultMessage：包装后的工具结果
    """
    tool_use_id = tool_block.id
    tool_name = tool_block.name
    tool_input = tool_block.input

    # === 1. 输入验证（可选）===
    if tool.input_schema:
        try:
            validated_input = tool.input_schema.model_validate(tool_input)
        except Exception as e:
            return _error_result(
                tool_use_id,
                f"Input validation error: {e}",
                is_recoverable=False,
            )

    # === 2. 权限检查（canUseTool 钩子）===
    # 注意：这里应该调用 hooks/canUseTool.py 进行权限检查
    # can_result = await canUseTool(tool, validated_input, context, assistant_message, tool_use_id)
    # if can_result.behavior != "allow":
    #     return ToolResultMessage(...)
    # 简化：直接执行

    # === 3. 执行工具 ===
    try:
        result = await tool.handler(
            validated_input or tool_input,
            context,
        )
    except asyncio.CancelledError:
        return _error_result(
            tool_use_id,
            "Tool execution cancelled",
            is_recoverable=False,
        )
    except Exception as e:
        # 工具执行异常
        return _error_result(
            tool_use_id,
            f"Tool execution error: {str(e)}",
            is_recoverable=True,
        )

    # === 4. 包装结果 ===
    return ToolResultMessage(
        content=[{
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": result.data if hasattr(result, "data") else result,
            "is_error": getattr(result, "is_error", False),
        }],
        tool_use_id=tool_use_id,
        is_error=getattr(result, "is_error", False),
    )


def _error_result(
    tool_use_id: str,
    message: str,
    is_recoverable: bool,
) -> ToolResultMessage:
    """构造错误结果"""
    return ToolResultMessage(
        content=[{
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": message,
            "is_error": True,
        }],
        tool_use_id=tool_use_id,
        is_error=True,
    )
```

### 5.4 ToolResult 包装返回

```python
# tools/tool.py
from pydantic import BaseModel
from typing import Any


class ToolResult(BaseModel):
    """Tool handler 返回结果"""
    data: Any  # 工具输出内容
    new_messages: list[Any] | None = None  # 可选：附带新消息（如 progress）
    context_modifier: Callable | None = None  # 可选：修改 context
    is_error: bool = False
    mcp_meta: dict | None = None  # MCP 协议元数据


# Tool handler 签名示例
"""
async def my_tool_handler(
    input: MyInputSchema,
    context: ToolUseContext,
) -> ToolResult:
    # 执行逻辑
    return ToolResult(data="操作完成")
"""
```

### 5.5 完整 tool_use 处理流程

```python
async def process_tool_uses(
    assistant_msg: AssistantMessage,
    context: ToolUseContext,
    registry: ToolRegistry,
) -> ToolResultMessage:
    """
    处理 AssistantMessage 中的所有 tool_use 块。

    流程：
    1. 提取 tool_use 块
    2. 根据 name 查找 Tool
    3. 构造 ToolUseContext
    4. 调用 tool.handler()
    5. 包装 ToolResultMessage
    """
    tool_blocks = extract_tool_uses(assistant_msg)
    results: list[ToolResultMessage] = []

    for block in tool_blocks:
        tool = registry.get(block.name)

        if tool is None:
            # 工具不存在
            results.append(_error_result(
                block.id,
                f"Tool '{block.name}' not found",
                is_recoverable=False,
            ))
            continue

        # 执行工具
        result = await execute_tool(
            tool_block=block,
            tool=tool,
            context=context,
            assistant_message=assistant_msg,
        )
        results.append(result)

    return results
```

---

## 6. Skill 触发机制

### 6.1 检测用户输入中的 skill trigger keyword

```python
# skills/trigger.py
import re
from typing import NamedTuple, Optional


class SkillTrigger(NamedTuple):
    """Skill 触发结果"""
    skill_name: str
    args: str  # 触发词后的参数


# trigger keywords 配置（可从配置文件加载）
SKILL_TRIGGERS: dict[str, str] = {
    "/simplify": "simplify",
    "/verify": "verify",
    "/brainstorm": "brainstorming",
    "/plan": "planning",
    "/review": "code-review",
    # 更多 trigger...
}

# 正则匹配：/triggerword 可选参数
TRIGGER_PATTERN = re.compile(r"^(/\w+)(.*)$")


def detect_skill_trigger(user_input: str) -> Optional[SkillTrigger]:
    """
    检测用户输入是否包含 skill trigger keyword。

    支持格式：
    - /simplify  # 无参数
    - /simplify refactor this code  # 带参数
    - /verify --fix  # 带标志

    Returns:
        SkillTrigger(name, args) 或 None
    """
    match = TRIGGER_PATTERN.match(user_input.strip())
    if not match:
        return None

    trigger_word = match.group(1)
    args = match.group(2).strip()

    skill_name = SKILL_TRIGGERS.get(trigger_word)
    if not skill_name:
        return None

    return SkillTrigger(skill_name=skill_name, args=args)
```

### 6.2 调用 Skill.get_prompt() 并注入消息列表

```python
# skills/skill.py
from pydantic import BaseModel
from typing import Any, Literal, Optional


class Skill(BaseModel):
    """Skill 定义（frontmatter 解析后的 in-memory 表示）"""
    name: str
    description: str
    user_invocable: bool = True
    allowed_tools: list[str] = []
    argument_hint: Optional[str] = None
    arguments: list[str] = []
    when_to_use: Optional[str] = None
    model: Optional[str] = None
    disable_model_invocation: bool = False
    context: Literal["inline", "fork"] | None = None
    agent: Optional[str] = None
    effort: Literal["low", "medium", "high"] | None = None
    paths: list[str] | None = None
    hooks: dict | None = None
    shell: dict | None = None
    markdown_content: str = ""
    base_dir: Optional[str] = None
    source: Literal["bundled", "user", "project", "managed"] = "bundled"

    async def get_prompt(
        self,
        args: str,
        tool_use_context: dict[str, Any],
    ) -> list[dict[str, Any]]:
        """
        返回发送给 LLM 的 prompt 内容块。
        执行变量替换：${CLAUDE_SKILL_DIR}、${CLAUDE_SESSION_ID}、${arg1} 等。
        """
        content = self.markdown_content

        # 变量替换
        if self.base_dir:
            content = content.replace("${CLAUDE_SKILL_DIR}", self.base_dir)

        session_id = tool_use_context.get("session_id", "unknown")
        content = content.replace("${CLAUDE_SESSION_ID}", session_id)

        # 参数替换
        args_list = args.split()
        for i, arg_name in enumerate(self.arguments):
            placeholder = f"${{{arg_name}}}"
            replacement = args_list[i] if i < len(args_list) else ""
            content = content.replace(placeholder, replacement)

        # 处理 inline shell 执行（!`...`）
        content = await self._process_shell_commands(content)

        return [{"type": "text", "text": content}]

    async def _process_shell_commands(self, content: str) -> str:
        """处理 inline shell 执行（!`...`）"""
        # 简化实现：实际应调用安全 shell 执行
        shell_pattern = re.compile(r"!`([^`]+)`")
        matches = shell_pattern.findall(content)
        for cmd in matches:
            # 安全检查...
            # result = await run_shell(cmd)
            # content = content.replace(f"!`{cmd}`", result)
            pass
        return content


# skills/registry.py
class SkillRegistry:
    """全局 Skill 注册表"""

    def __init__(self):
        self._skills: dict[str, Skill] = {}

    def register(self, skill: Skill) -> None:
        """注册 skill"""
        self._skills[skill.name] = skill

    def get(self, name: str) -> Optional[Skill]:
        """根据名称获取 skill"""
        return self._skills.get(name)

    def trigger(self, user_input: str) -> tuple[Skill, str] | None:
        """
        尝试从用户输入触发 skill。

        Returns:
            (Skill, args) 或 None
        """
        trigger_result = detect_skill_trigger(user_input)
        if not trigger_result:
            return None

        skill = self.get(trigger_result.skill_name)
        if not skill:
            return None

        return (skill, trigger_result.args)
```

### 6.3 Skill 注入消息列表

```python
# agent/loop.py 中处理 skill 触发
from agent.types import HumanMessage


async def process_skill_trigger(
    user_input: str,
    skill_registry: SkillRegistry,
    tool_use_context: ToolUseContext,
) -> list[HumanMessage] | None:
    """
    处理用户输入中的 skill trigger。

    Returns:
        要添加到消息列表的新消息，或 None（无 trigger）
    """
    result = skill_registry.trigger(user_input)
    if not result:
        return None

    skill, args = result

    # 获取 skill prompt
    prompt_blocks = await skill.get_prompt(args, tool_use_context)

    # 构造 HumanMessage
    # 注意：skill 输出通常作为 system message 或 assistant message 注入
    # 具体取决于 skill.context 配置（inline 或 fork）
    if skill.context == "fork":
        # fork 模式：作为独立 assistant message
        return [
            HumanMessage(
                content=prompt_blocks[0]["text"] if prompt_blocks else "",
                is_meta=True,  # 标记为元消息
            )
        ]
    else:
        # inline 模式：直接作为 text block 追加
        return [
            HumanMessage(
                content=prompt_blocks[0]["text"] if prompt_blocks else "",
            )
        ]
```

### 6.4 trigger 匹配优先级

```python
# 优先级规则（按顺序）
PRIORITY_RULES = [
    # 1. 精确匹配：trigger word 完全相等
    lambda trigger: trigger.trigger_word in SKILL_TRIGGERS,

    # 2. 别名匹配：trigger word 是某个 skill 的别名
    # (在 SkillRegistry 中通过 aliases 字段处理)

    # 3. 模糊匹配：包含 trigger word（最后兜底）
    # 不实现模糊匹配以避免误触发
]

# 示例：多个 skill 竞争同一 trigger 时的处理
CONFLICT_RESOLUTION = {
    # 同名处理：bundled < user < project < managed
    # 后注册的覆盖先注册的
}
```

---

## 7. 错误处理

### 7.1 LLM 调用失败的重试策略

```python
# agent/retry.py
import asyncio
from typing import TypeVar, Callable, Awaitable
from dataclasses import dataclass


T = TypeVar("T")


@dataclass
class RetryConfig:
    """重试配置"""
    max_retries: int = 3
    initial_delay_ms: int = 1000
    max_delay_ms: int = 30000
    exponential_base: float = 2.0
    jitter: bool = True


class RetryableError(Exception):
    """可重试的错误基类"""
    pass


class RateLimitError(RetryableError):
    """速率限制错误"""
    retry_after_ms: int | None = None


class PromptTooLongError(RetryableError):
    """Prompt 太长错误"""
    pass


class APIError(RetryableError):
    """通用 API 错误"""
    status_code: int | None = None


async def with_retry(
    func: Callable[..., Awaitable[T]],
    config: RetryConfig | None = None,
    *args,
    **kwargs,
) -> T:
    """
    带重试的函数调用。

    使用指数退避算法（exponential backoff），
    支持 jitter 以避免惊群效应（thundering herd）。
    """
    config = config or RetryConfig()
    last_exception: Exception | None = None

    for attempt in range(config.max_retries + 1):
        try:
            return await func(*args, **kwargs)

        except RateLimitError as e:
            # 速率限制：使用 server 返回的 retry_after_ms
            delay_ms = e.retry_after_ms or config.initial_delay_ms
            last_exception = e

        except PromptTooLongError:
            # Prompt 太长：无法通过重试解决，向上传播
            raise

        except APIError as e:
            # 其他 API 错误：指数退避
            if attempt < config.max_retries:
                delay_ms = min(
                    config.initial_delay_ms * (config.exponential_base ** attempt),
                    config.max_delay_ms,
                )
                last_exception = e
            else:
                raise

        except Exception as e:
            # 非 RetryableError：直接向上传播
            raise

        # 等待后重试
        if config.jitter:
            import random
            delay_ms = int(delay_ms * (0.5 + random.random()))

        await asyncio.sleep(delay_ms / 1000)

    raise last_exception


# 使用示例
async def call_llm_with_retry(
    query_engine: QueryEngine,
    messages: list[dict],
) -> Any:
    """带重试的 LLM 调用"""
    async def _call():
        async for event in query_engine.submit_message(messages):
            if event.type == "assistant":
                return event
        raise APIError("No assistant message received")

    return await with_retry(_call)
```

### 7.2 Tool 执行异常的捕获和上报

```python
# agent/tool_execution.py
import traceback
from agent.types import ToolResultMessage

# ToolExecutionError 和 ValidationError 的定义见第 5.3 节（agent/tool_execution.py）。
# 此处直接使用，不重复定义：
# from agent.tool_execution import ToolExecutionError, ValidationError


async def execute_tool_safe(
    tool_block: ToolUseBlock,
    tool: Tool,
    context: ToolUseContext,
) -> ToolResultMessage:
    """
    安全执行工具，捕获并包装所有异常。
    """
    tool_use_id = tool_block.id

    try:
        result = await tool.handler(tool_block.input, context)
        return _wrap_tool_result(result, tool_use_id)

    except ToolExecutionError as e:
        # 已知的工具执行错误
        return _error_result(
            tool_use_id,
            str(e),
            is_recoverable=e.is_recoverable,
        )

    except ValidationError as e:
        # Pydantic 验证错误
        return _error_result(
            tool_use_id,
            f"Input validation error: {e}",
            is_recoverable=False,  # 输入错误不重试
        )

    except asyncio.CancelledError:
        # 外部取消
        return _error_result(
            tool_use_id,
            "Tool execution cancelled",
            is_recoverable=False,
        )

    except Exception as e:
        # 未知错误：记录 traceback，上报
        error_msg = f"Unexpected error: {type(e).__name__}: {str(e)}"

        # 记录详细错误（用于调试）
        print(f"[ERROR] Tool {tool_block.name} failed:")
        traceback.print_exc()

        return _error_result(
            tool_use_id,
            error_msg,
            is_recoverable=True,  # 假设可恢复
        )


def _error_result(
    tool_use_id: str,
    message: str,
    is_recoverable: bool,
) -> ToolResultMessage:
    """构造错误结果"""
    return ToolResultMessage(
        content=[{
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": message,
            "is_error": True,
        }],
        tool_use_id=tool_use_id,
        is_error=True,
    )


def _wrap_tool_result(result: Any, tool_use_id: str) -> ToolResultMessage:
    """包装 Tool handler 返回结果"""
    if hasattr(result, "data"):
        content = result.data
        is_error = getattr(result, "is_error", False)
    else:
        content = result
        is_error = False

    return ToolResultMessage(
        content=[{
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": content,
            "is_error": is_error,
        }],
        tool_use_id=tool_use_id,
        is_error=is_error,
    )
```

### 7.3 会话级错误的处理

```python
# agent/loop.py 中会话级错误处理
import uuid


async def run_agent_loop_with_error_handling(
    initial_messages: list[HumanMessage | AssistantMessage],
    config: LoopConfig,
) -> AsyncGenerator[Any, AssistantMessage | None]:
    """
    封装 run_agent_loop，添加会话级错误处理。
    """
    errors: list[str] = []  # 错误日志

    try:
        async for msg in run_agent_loop(initial_messages, config):
            yield msg

    except asyncio.CancelledError:
        # 用户中断：返回中断前的最后状态
        yield SystemMessage(
            type="system",
            subtype="user_interrupt",
            content="Interrupted by user",
        )
        return None

    except Exception as e:
        # 未捕获的异常：记录并返回错误消息
        error_id = str(uuid.uuid4())
        error_msg = f"[Error {error_id}] {type(e).__name__}: {str(e)}"

        yield SystemMessage(
            type="system",
            subtype="session_error",
            content=error_msg,
        )

        # 尝试恢复：注入错误消息，继续循环
        # （如果配置允许）
        if config.max_turns and config.max_turns > 0:
            # 注入错误消息，继续一轮
            recovery_msg = HumanMessage(
                content=f"Session error occurred: {error_msg}. Please continue.",
                is_meta=True,
            )
            # 递归调用（限制深度）
            # ... 省略实现
        else:
            return None
```

---

## 8. 关键接口契约

### 8.1 run_agent_loop() 函数签名和返回值

```python
# agent/loop.py


async def run_agent_loop(
    initial_messages: list[HumanMessage | AssistantMessage],
    config: LoopConfig,
) -> AsyncGenerator[
    # Yields:
    AssistantMessage | ToolResultMessage | SystemMessage | StreamEvent,
    # Returns:
    AssistantMessage | None,
]:
    """
    Agent 主循环。

    参数:
        initial_messages: 初始消息列表。
            通常只有一条 HumanMessage（用户输入）。
            可包含之前的对话历史用于恢复（resume）。

        config: LoopConfig
            - tools: 可用工具列表
            - max_turns: 最大轮次限制（None=无限制）
            - max_budget_usd: 最大 USD 预算（None=无限制）
            - task_budget: API 内部 task budget 配置
            - system_prompt: 系统提示词

    Yields:
        AssistantMessage:
            - 增量输出（当 stop_reason=None 或 tool_use）
            - 最终输出（当 stop_reason=end_turn/stop_sequence）

        ToolResultMessage:
            - 工具执行结果（role=user，携带 tool_use_id）

        SystemMessage:
            - api_retry: API 重试提示
            - max_turns_reached: 轮次耗尽
            - max_budget_reached: 预算耗尽
            - user_interrupt: 用户中断
            - session_error: 会话错误

        StreamEvent:
            - 仅当 config.include_partial_messages=True
            - SDK 原始流式事件

    Returns:
        AssistantMessage | None:
            - 正常结束：返回最后一条 AssistantMessage
            - 异常结束：返回 None
    """
    ...
```

### 8.2 QueryEngine.create() 配置项

```python
# agent/query_engine.py


@dataclass
class QueryEngineConfig:
    """
    QueryEngine 构造配置。

    所有字段均有默认值，但以下字段必须设置：
    - tools
    """
    # === 必需字段 ===
    tools: list[Any] = field(default_factory=list)

    # === 可选字段 ===
    system_prompt: str = ""
    max_turns: int | None = None
    max_budget_usd: float | None = None
    task_budget: dict | None = None
    json_schema: dict | None = None
    verbose: bool = False
    include_partial_messages: bool = False

    # === 回调 ===
    on_usage: Callable[[dict], None] | None = None
    on_status: Callable[[str], None] | None = None

    # === 内部字段（通常由 loop.py 设置）===
    tool_use_context: dict | None = None
    abort_controller: asyncio.Event | None = None
    orphaned_permission: dict | None = None

    # === SDK 特有配置 ===
    user_specified_model: str | None = None
    fallback_model: str | None = None
    thinking_config: dict | None = None


class QueryEngine:
    @classmethod
    def create(cls, config: QueryEngineConfig) -> "QueryEngine":
        """
        工厂方法：创建 QueryEngine 实例。
        """
        return cls(config)

    # 或直接构造
    def __init__(self, config: QueryEngineConfig):
        ...
```

### 8.3 ToolUseContext 完整字段列表

```python
# tools/tool.py


@dataclass
class ToolUseContext:
    """
    Tool 执行上下文。

    包含工具执行所需的所有运行时信息。
    """
    # === options（工具选项）===
    options: ToolUseOptions

    # === 中断控制 ===
    abort_controller: asyncio.Event
    """
    用于取消正在执行的工具。
    设置事件时表示请求取消。
    """

    # === 状态管理 ===
    get_app_state: Callable[[], Any]
    """获取当前应用状态（只读）"""

    set_app_state: Callable[[Callable[[Any], Any]], None]
    """
    更新应用状态。
    参数是纯函数：(prev AppState) -> AppState
    """

    # === 消息历史 ===
    messages: list[Any]
    """
    当前对话的所有消息。
    Tool 可以读取历史来理解上下文。
    """

    # === 查询追踪 ===
    query_tracking: dict | None = None
    """
    用于追踪嵌套 agent 调用链。
    - chain_id: 调用链 ID
    - depth: 当前深度（0=主循环）
    """

    # === 文件状态缓存 ===
    read_file_state: dict | None = None
    """
    文件读取缓存。
    Tool 可以查询文件状态。
    """

    # === 进度回调 ===
    set_in_progress_tool_use_ids: Callable[[Callable[[set], set]], None]
    """
    更新当前正在执行的 tool_use_id 集合。
    用于 UI 显示进度。
    """

    set_response_length: Callable[[Callable[[int], int]], None]
    """
    设置预期响应长度（token 数）。
    用于进度估算。
    """

    # === 工具决策缓存 ===
    tool_decisions: dict | None = None
    """
    工具权限决策缓存。
    key: tool_name
    value: {source, decision, timestamp}
    """

    # === Skill 相关 ===
    discovered_skill_names: set[str] | None = None
    """
    本会话中发现的 skill 名称集合。
    用于遥测（was_discovered）。
    """

    dynamic_skill_dir_triggers: set[str] | None = None
    nested_memory_attachment_triggers: set[str] | None = None

    # === 子 agent 信息 ===
    agent_id: str | None = None
    """子 agent ID（仅子 agent 设置）"""

    agent_type: str | None = None
    """子 agent 类型名"""

    # === 扩展钩子 ===
    context_modifier: Callable[["ToolUseContext"], "ToolUseContext"] | None = None
    """
    可选：返回修改后的 context。
    用于工具动态修改上下文。
    """


@dataclass
class ToolUseOptions:
    """ToolUseContext.options"""
    tools: list[Any] = field(default_factory=list)
    main_loop_model: str = "claude-sonnet-4-20250514"
    thinking_config: dict = field(default_factory=lambda: {"type": "disabled"})
    is_non_interactive_session: bool = True
    max_budget_usd: float | None = None
    custom_system_prompt: str | None = None
    append_system_prompt: str | None = None
    query_source: str | None = None
```

### 8.4 类型使用示例

```python
# 使用示例
async def example_tool_handler(
    input: dict,
    context: ToolUseContext,
) -> ToolResult:
    """Tool handler 示例"""

    # 读取消息历史
    last_user_msg = None
    for msg in reversed(context.messages):
        if hasattr(msg, "role") and msg.role == "user":
            last_user_msg = msg
            break

    # 获取应用状态
    app_state = context.get_app_state()

    # 检查是否应取消
    if context.abort_controller.is_set():
        raise asyncio.CancelledError()

    # 更新状态（如果需要）
    def update(state: Any) -> Any:
        return {**state, "tool_called": True}
    context.set_app_state(update)

    # 返回结果
    return ToolResult(data={"status": "ok"})
```

---

## 附录 A: 与 TypeScript 实现的映射关系

| TypeScript (src/query.ts) | Python (agent/) | 说明 |
|---------------------------|-----------------|------|
| `query()` | `run_agent_loop()` | 主循环函数 |
| `QueryEngine.submitMessage()` | `QueryEngine.submit_message()` | SDK 调用封装 |
| `while(true)` | `while True` | 主循环 |
| `State` | `LoopState` | 循环状态 |
| `QueryEngineConfig` | `QueryEngineConfig` | 配置类 |
| `toolUseBlocks` | `ToolUseBlock` | tool_use 块 |
| `canUseTool` | - | 权限钩子（简化实现） |
| `stop_reason` | `stop_reason` | 停止原因 |
| `message_delta` | SDK event | 流式事件 |

## 附录 B: 参考文件

- 源实现：`src/query.ts`
- SDK 封装：`src/QueryEngine.ts`
- Tool 接口：`src/Tool.ts`
- 架构概览：`docs/agent-architecture.md`
