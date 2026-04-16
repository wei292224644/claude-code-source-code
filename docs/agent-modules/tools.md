# Tools 模块技术文档

> 参考来源: TypeScript 源码 `src/Tool.ts` (Tool 接口 + `buildTool` 工厂) 与 `src/tools.ts` (注册表)
>
> 本文档描述 Python 实现方案，框架要求: `claude-agent-sdk>=0.1.59`

---

## 1. Tool 接口设计 (`tools/tool.py`)

### 1.1 dataclass Tool 的完整定义

```python
# tools/tool.py
from __future__ import annotations

import threading
from dataclasses import dataclass, field
from typing import (
    Any,
    Callable,
    Generic,
    TypeVar,
)

from pydantic import BaseModel, Field
from typing_extensions import TypedDict


# ─────────────────────────────────────────────────────────────
# Input Schema 类型
# ─────────────────────────────────────────────────────────────

class ToolInputJSONSchema(TypedDict):
    type: str
    properties: dict[str, Any] | None


# ─────────────────────────────────────────────────────────────
# ToolUseContext — 调用上下文字面量类型
# ─────────────────────────────────────────────────────────────

class ToolUseContextOptions(TypedDict, total=False):
    commands: list[Any]
    debug: bool
    mainLoopModel: str
    tools: list[Any]
    verbose: bool
    thinkingConfig: dict[str, Any]
    mcpClients: list[Any]
    mcpResources: dict[str, list[Any]]
    isNonInteractiveSession: bool
    agentDefinitions: dict[str, Any]
    maxBudgetUsd: float | None
    customSystemPrompt: str | None
    appendSystemPrompt: str | None
    querySource: str | None
    refreshTools: Callable[[], list[Any]] | None


class ToolUseContext(TypedDict):
    """Tool 调用的完整上下文对象."""
    options: ToolUseContextOptions
    abortController: Any  # AbortController
    readFileState: Any
    getAppState: Callable[[], dict[str, Any]]
    setAppState: Callable[[Callable[[dict[str, Any]], dict[str, Any]]], None]
    setAppStateForTasks: Callable[[Callable[[dict[str, Any]], dict[str, Any]]], None] | None
    handleElicitation: Callable[..., Any] | None
    setToolJSX: Callable[..., None] | None
    addNotification: Callable[[dict[str, Any]], None] | None
    appendSystemMessage: Callable[[dict[str, Any]], None] | None
    sendOSNotification: Callable[[dict[str, Any]], None] | None
    nestedMemoryAttachmentTriggers: set[str] | None
    loadedNestedMemoryPaths: set[str] | None
    dynamicSkillDirTriggers: set[str] | None
    discoveredSkillNames: set[str] | None
    userModified: bool | None
    setInProgressToolUseIDs: Callable[[Callable[[set[str]], set[str]]], None]
    setHasInterruptibleToolInProgress: Callable[[bool], None] | None
    setResponseLength: Callable[[Callable[[int], int]], None]
    pushApiMetricsEntry: Callable[[float], None] | None
    setStreamMode: Callable[[str], None] | None
    onCompactProgress: Callable[[dict[str, Any]], None] | None
    setSDKStatus: Callable[[dict[str, Any]], None] | None
    openMessageSelector: Callable[[], None] | None
    updateFileHistoryState: Callable[[Callable[[dict[str, Any]], dict[str, Any]]], None]
    updateAttributionState: Callable[[Callable[[dict[str, Any]], dict[str, Any]]], None]
    setConversationId: Callable[[str], None] | None
    agentId: str | None
    agentType: str | None
    requireCanUseTool: bool | None
    messages: list[Any]
    fileReadingLimits: dict[str, Any] | None
    globLimits: dict[str, Any] | None
    toolDecisions: dict[str, dict[str, Any]] | None
    queryTracking: dict[str, Any] | None
    requestPrompt: Callable[[str, str | None], Callable[..., Any]] | None
    toolUseId: str | None
    criticalSystemReminder_EXPERIMENTAL: str | None
    preserveToolUseResults: bool | None
    localDenialTracking: dict[str, Any] | None
    contentReplacementState: dict[str, Any] | None
    renderedSystemPrompt: dict[str, Any] | None


# ─────────────────────────────────────────────────────────────
# ValidationResult — 输入验证结果
# ─────────────────────────────────────────────────────────────

class ValidationResult(TypedDict):
    result: True

class ValidationResultError(TypedDict):
    result: False
    message: str
    errorCode: int


# ─────────────────────────────────────────────────────────────
# ToolResult — 工具执行结果（泛型容器）
# ─────────────────────────────────────────────────────────────

@dataclass
class ToolResult(Generic[T]):
    """Tool.call() 的返回值封装."""
    data: T
    new_messages: list[Any] | None = None
    context_modifier: Callable[[ToolUseContext], ToolUseContext] | None = None
    mcp_meta: dict[str, Any] | None = None


# ─────────────────────────────────────────────────────────────
# ToolProtocol — 工具对象必须实现的接口
# ─────────────────────────────────────────────────────────────

class ToolProtocol:
    """
    Tool 对象必须实现的接口。
    对应 TypeScript Tool<Input, Output, P> 类型。
    """

    name: str
    maxResultSizeChars: int

    @property
    def input_schema(self) -> type[BaseModel]:
        raise NotImplementedError

    async def call(
        self,
        args: dict[str, Any],
        context: ToolUseContext,
        can_use_tool: Callable[..., Any] | None = None,
        parent_message: dict[str, Any] | None = None,
        on_progress: Callable[[dict[str, Any]], None] | None = None,
    ) -> ToolResult[Any]:
        raise NotImplementedError

    async def description(self, input_args: dict[str, Any]) -> str:
        raise NotImplementedError

    def is_enabled(self) -> bool:
        raise NotImplementedError

    def is_concurrency_safe(self, input_args: dict[str, Any]) -> bool:
        raise NotImplementedError

    def is_read_only(self, input_args: dict[str, Any]) -> bool:
        raise NotImplementedError

    def is_destructive(self, input_args: dict[str, Any]) -> bool:
        raise NotImplementedError

    def user_facing_name(self, input_args: dict[str, Any] | None = None) -> str:
        raise NotImplementedError

    def to_auto_classifier_input(self, input_args: dict[str, Any]) -> str:
        raise NotImplementedError

    def map_tool_result_to_block_param(
        self, content: Any, tool_use_id: str
    ) -> dict[str, Any]:
        raise NotImplementedError
```

### 1.2 字段含义说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `str` | 工具唯一标识名，如 `"web_fetch"`, `"bash"` |
| `description` | `async def(input) -> str` | 动态生成工具描述（根据输入参数），用于提示词注入 |
| `inputSchema` | `type[BaseModel]` | Pydantic 模型，定义工具输入参数的 schema，同时用于 API 验证 |
| `maxResultSizeChars` | `int` | 结果阈值；超过此大小时结果持久化到磁盘，API 收到预览路径 |
| `isEnabled` | `def() -> bool` | 运行时开关（如环境变量控制） |
| `isReadOnly` | `def(input) -> bool` | 标识工具是否只读（影响 UI 展示） |
| `isDestructive` | `def(input) -> bool` | 标识不可逆操作（删除、覆写），影响确认提示 |
| `validateInput` | `async def(input, ctx) -> ValidationResult` | 调用前校验参数，返回结构化错误 |
| `checkPermissions` | `async def(input, ctx) -> PermissionResult` | 权限检查，返回 `allow/deny/ask` |
| `isConcurrencySafe` | `def(input) -> bool` | 是否可并发执行（影响并行 tool_use 调度） |
| `toAutoClassifierInput` | `def(input) -> str` | 安全分类器的输入摘要 |
| `renderToolUseMessage` | `React 组件` | 工具调用时的 UI 渲染（TypeScript 特有） |

### 1.3 ToolUseContext 包含的上下文字段

关键字段说明：

- **`getAppState()` / `setAppState()`** — 访问/修改全局应用状态（如 `mcp.tools`、`toolPermissionContext`）
- **`abortController`** — 标准 `AbortController`，用于取消正在执行的工具
- **`options.mcpClients`** — 已连接的 MCP 服务器列表，MCP 工具调用时使用
- **`options.tools`** — 当前可用的工具列表
- **`messages`** — 当前会话的消息历史
- **`toolUseId`** — 当前 tool_use 块的唯一 ID

---

## 2. `build_tool` 工厂函数

### 2.1 完整 Python 实现

```python
# tools/factory.py
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Callable, TypeVar

from pydantic import BaseModel, ValidationError

from .tool import (
    ToolProtocol,
    ToolResult,
    ToolUseContext,
)


T = TypeVar("T")


@dataclass
class ToolDefaults:
    """build_tool 提供的默认值（对应 TypeScript TOOL_DEFAULTS）."""
    is_enabled: Callable[[], bool] = field(default_factory=lambda: lambda: True)
    is_concurrency_safe: Callable[[Any], bool] = field(default_factory=lambda _: False)
    is_read_only: Callable[[Any], bool] = field(default_factory=lambda _: False)
    is_destructive: Callable[[Any], bool] = field(default_factory=lambda _: False)
    check_permissions: Callable[..., dict[str, Any]] = field(
        default_factory=lambda input, ctx=None: {"behavior": "allow", "updatedInput": input}
    )
    to_auto_classifier_input: Callable[[Any], str] = field(default_factory=lambda _: "")
    user_facing_name: Callable[[Any | None], str] = field(default_factory=lambda _: "")


TOOL_DEFAULTS = ToolDefaults()


class ToolDefError(Exception):
    """Tool 定义不完整或验证失败时抛出."""
    def __init__(self, message: str, error_code: int = 1):
        self.message = message
        self.error_code = error_code
        self.validation_result: dict[str, Any] | None = None
        super().__init__(message)


def build_tool(
    name: str,
    input_schema: type[BaseModel],
    call: Callable[
        [dict[str, Any], ToolUseContext, Any | None, Any | None, Any | None],
        ToolResult[Any],
    ],
    output_schema: type[BaseModel] | None = None,
    max_result_size_chars: int = 100_000,
    description: Callable[[dict[str, Any]], str] | str | None = None,
    prompt: Callable[[dict[str, Any]], str] | str | None = None,
    is_enabled: Callable[[], bool] | None = None,
    is_concurrency_safe: Callable[[dict[str, Any]], bool] | None = None,
    is_read_only: Callable[[dict[str, Any]], bool] | None = None,
    is_destructive: Callable[[dict[str, Any]], bool] | None = None,
    validate_input: Callable[
        [dict[str, Any], ToolUseContext], dict[str, Any]
    ] | None = None,
    check_permissions: Callable[
        [dict[str, Any], ToolUseContext], dict[str, Any]
    ] | None = None,
    to_auto_classifier_input: Callable[[dict[str, Any]], str] | None = None,
    user_facing_name: Callable[[dict[str, Any] | None], str] | None = None,
    get_tool_use_summary: Callable[[dict[str, Any]], str | None] | None = None,
    get_activity_description: Callable[
        [dict[str, Any]], str | None
    ] | None = None,
    render_tool_result_message: Any = None,
    render_tool_use_message: Any = None,
    render_tool_use_progress_message: Any = None,
    is_result_truncated: Callable[[Any], bool] | None = None,
    interrupt_behavior: Callable[[], str] | None = None,
    is_search_or_read_command: Callable[
        [dict[str, Any]], dict[str, bool]
    ] | None = None,
    is_open_world: Callable[[dict[str, Any]], bool] | None = None,
    requires_user_interaction: Callable[[], bool] | None = None,
    backfill_observable_input: Callable[[dict[str, Any]], None] | None = None,
    get_path: Callable[[dict[str, Any]], str | None] | None = None,
    prepare_permission_matcher: Callable[
        [dict[str, Any]], Callable[[str], bool]
    ] | None = None,
    aliases: list[str] | None = None,
    search_hint: str | None = None,
    should_defer: bool = False,
    always_load: bool = False,
    strict: bool = False,
    is_mcp: bool = False,
    is_lsp: bool = False,
    mcp_info: dict[str, str] | None = None,
    **extra: Any,
) -> ToolProtocol:
    """
    build_tool 工厂函数 — 对应 TypeScript buildTool<D>(def: D)。

    用法示例:
        WebFetchTool = build_tool(
            name="web_fetch",
            input_schema=WebFetchInput,
            call=web_fetch_handler,
            max_result_size_chars=100_000,
            should_defer=True,
        )
    """
    _input_schema = input_schema

    # ── 输入验证器 ──────────────────────────────────────────

    def _validate_args(args: dict[str, Any]) -> dict[str, Any]:
        """使用 Pydantic schema 做输入验证，失败时抛出 ToolDefError."""
        try:
            validated = _input_schema.model_validate(args)
            return validated.model_dump()
        except ValidationError as e:
            errors = e.errors()
            msg = "; ".join(
                ".".join(str(loc) for loc in err["loc"]) + ": " + err["msg"]
                for err in errors
            )
            raise ToolDefError(
                f"Input validation failed for tool '{name}': {msg}",
                error_code=1,
            )

    # ── 构建 call 方法 ──────────────────────────────────────

    _call_impl = call

    async def _call(
        args: dict[str, Any],
        context: ToolUseContext,
        can_use_tool: Any | None = None,
        parent_message: Any | None = None,
        on_progress: Any | None = None,
    ) -> ToolResult[Any]:
        # Pydantic schema 验证
        validated_args = _validate_args(args)

        # validateInput 钩子
        if validate_input is not None:
            result = validate_input(validated_args, context)
            if not result.get("result", True):
                err = ToolDefError(
                    result.get("message", "Validation failed"),
                    error_code=result.get("errorCode", 1),
                )
                err.validation_result = result
                raise err

        # 调用实际 handler
        if _call_impl is None:
            return ToolResult(data=None)

        return await _call_impl(
            validated_args,
            context,
            can_use_tool,
            parent_message,
            on_progress,
        )

    # ── 构建 description 方法 ──────────────────────────────

    if callable(description):
        _description_impl = description
    elif description is not None:
        _description_impl = lambda input, **_: str(description)  # type: ignore
    else:
        _description_impl = lambda input, **_: f"Tool: {name}"  # type: ignore

    async def _description(input_args: dict[str, Any]) -> str:
        return _description_impl(input_args)

    # ── 组装 Tool 对象 ─────────────────────────────────────

    return _ToolImpl(
        name=name,
        input_schema=_input_schema,
        output_schema=output_schema,
        max_result_size_chars=max_result_size_chars,
        description=_description_impl,
        prompt=prompt,
        is_enabled=is_enabled or TOOL_DEFAULTS.is_enabled,
        is_concurrency_safe=is_concurrency_safe or TOOL_DEFAULTS.is_concurrency_safe,
        is_read_only=is_read_only or TOOL_DEFAULTS.is_read_only,
        is_destructive=is_destructive or TOOL_DEFAULTS.is_destructive,
        validate_input=validate_input,
        check_permissions=check_permissions or TOOL_DEFAULTS.check_permissions,
        to_auto_classifier_input=to_auto_classifier_input or TOOL_DEFAULTS.to_auto_classifier_input,
        user_facing_name=user_facing_name or (lambda inp: name),
        get_tool_use_summary=get_tool_use_summary,
        get_activity_description=get_activity_description,
        render_tool_result_message=render_tool_result_message,
        render_tool_use_message=render_tool_use_message,
        render_tool_use_progress_message=render_tool_use_progress_message,
        is_result_truncated=is_result_truncated,
        interrupt_behavior=interrupt_behavior,
        is_search_or_read_command=is_search_or_read_command,
        is_open_world=is_open_world,
        requires_user_interaction=requires_user_interaction,
        backfill_observable_input=backfill_observable_input,
        get_path=get_path,
        prepare_permission_matcher=prepare_permission_matcher,
        aliases=aliases or [],
        search_hint=search_hint,
        should_defer=should_defer,
        always_load=always_load,
        strict=strict,
        is_mcp=is_mcp,
        is_lsp=is_lsp,
        mcp_info=mcp_info,
        _call=_call,
        **extra,
    )


# ─── 内部 Tool 实现类 ───────────────────────────────────────

class _ToolImpl:
    """
    build_tool 返回的完整 Tool 对象。
    实现了 ToolProtocol，供 Query Engine 调用。
    """

    def __init__(
        self,
        name: str,
        input_schema: type[BaseModel],
        output_schema: type[BaseModel] | None,
        max_result_size_chars: int,
        description: Callable[..., str],
        prompt: Any,
        is_enabled: Callable[[], bool],
        is_concurrency_safe: Callable[[Any], bool],
        is_read_only: Callable[[Any], bool],
        is_destructive: Callable[[Any], bool],
        validate_input: Any,
        check_permissions: Callable[..., dict[str, Any]],
        to_auto_classifier_input: Callable[[Any], str],
        user_facing_name: Callable[[Any | None], str],
        get_tool_use_summary: Any,
        get_activity_description: Any,
        render_tool_result_message: Any,
        render_tool_use_message: Any,
        render_tool_use_progress_message: Any,
        is_result_truncated: Any,
        interrupt_behavior: Any,
        is_search_or_read_command: Any,
        is_open_world: Any,
        requires_user_interaction: Any,
        backfill_observable_input: Any,
        get_path: Any,
        prepare_permission_matcher: Any,
        aliases: list[str],
        search_hint: str | None,
        should_defer: bool,
        always_load: bool,
        strict: bool,
        is_mcp: bool,
        is_lsp: bool,
        mcp_info: dict[str, str] | None,
        _call: Callable[..., ToolResult[Any]],
        **extra: Any,
    ):
        self.name = name
        self.input_schema = input_schema
        self.output_schema = output_schema
        self.max_result_size_chars = max_result_size_chars
        self.description = description
        self.prompt = prompt
        self.is_enabled = is_enabled
        self.is_concurrency_safe = is_concurrency_safe
        self.is_read_only = is_read_only
        self.is_destructive = is_destructive
        self.validate_input = validate_input
        self.check_permissions = check_permissions
        self.to_auto_classifier_input = to_auto_classifier_input
        self.user_facing_name = user_facing_name
        self.get_tool_use_summary = get_tool_use_summary
        self.get_activity_description = get_activity_description
        self.render_tool_result_message = render_tool_result_message
        self.render_tool_use_message = render_tool_use_message
        self.render_tool_use_progress_message = render_tool_use_progress_message
        self.is_result_truncated = is_result_truncated
        self.interrupt_behavior = interrupt_behavior
        self.is_search_or_read_command = is_search_or_read_command
        self.is_open_world = is_open_world
        self.requires_user_interaction = requires_user_interaction
        self.backfill_observable_input = backfill_observable_input
        self.get_path = get_path
        self.prepare_permission_matcher = prepare_permission_matcher
        self.aliases = aliases
        self.search_hint = search_hint
        self.should_defer = should_defer
        self.always_load = always_load
        self.strict = strict
        self.is_mcp = is_mcp
        self.is_lsp = is_lsp
        self.mcp_info = mcp_info
        self._call = _call
        self.__dict__.update(extra)

    async def call(
        self,
        args: dict[str, Any],
        context: ToolUseContext,
        can_use_tool: Any | None = None,
        parent_message: Any | None = None,
        on_progress: Any | None = None,
    ) -> ToolResult[Any]:
        return await self._call(args, context, can_use_tool, parent_message, on_progress)

    async def description(self, input_args: dict[str, Any]) -> str:
        return await self.description(input_args)  # type: ignore

    def map_tool_result_to_block_param(self, content: Any, tool_use_id: str) -> dict[str, Any]:
        """将执行结果映射为 SDK 的 ToolResultBlockParam 格式."""
        is_error = isinstance(content, dict) and content.get("error") is True
        return {
            "tool_use_id": tool_use_id,
            "type": "tool_result",
            "content": content if isinstance(content, str) else content.get("message", str(content)),
            "is_error": is_error or None,
        }
```

### 2.2 使用 Pydantic Schema 做输入验证

```python
from pydantic import BaseModel, Field
from typing import Annotated

class WebFetchInput(BaseModel):
    url: Annotated[str, Field(description="The URL to fetch content from")]
    prompt: Annotated[str, Field(description="The prompt to run on the fetched content")]

WebFetchTool = build_tool(
    name="web_fetch",
    input_schema=WebFetchInput,
    call=web_fetch_handler,
    max_result_size_chars=100_000,
    should_defer=True,
)
```

`build_tool` 内部在 `call()` 执行前自动调用 `_validate_args(args)`，使用 Pydantic 的 `model_validate` 验证输入。

### 2.3 错误处理（schema 验证失败时返回结构化错误 vs 抛出异常）

**Schema 验证失败 — 返回结构化错误（不抛出异常）：**

```python
def _validate_args(args):
    try:
        validated = _input_schema.model_validate(args)
        return validated.model_dump()
    except ValidationError as e:
        raise ToolDefError(
            f"Input validation failed: {e}",
            error_code=1,
        )
```

**在 `_call()` 中捕获并转换为 ToolResult 返回：**

```python
async def _call(...):
    try:
        validated_args = _validate_args(args)
    except ToolDefError as e:
        # 返回结构化错误，而非抛出
        return ToolResult(
            data={
                "error": True,
                "message": e.message,
                "errorCode": e.error_code,
            }
        )

    # validateInput 钩子失败
    if validate_input is not None:
        result = validate_input(validated_args, context)
        if not result.get("result", True):
            return ToolResult(
                data={
                    "error": True,
                    "message": result.get("message", "Validation failed"),
                    "errorCode": result.get("errorCode", 1),
                }
            )
```

**工具 handler 内部异常（如网络超时）：同样返回错误结果，不抛出：**

```python
async def web_fetch_handler(args, context, ...):
    try:
        response = await client.get(url)
    except httpx.TimeoutException:
        return ToolResult(
            data={
                "error": True,
                "message": f"Request timed out after 30s",
                "errorCode": 2,
            }
        )
```

---

## 3. ToolResult 格式

### 3.1 Pydantic 模型定义

```python
# tools/result.py
from __future__ import annotations

from typing import Any, Generic, TypeVar

from pydantic import BaseModel


T = TypeVar("T")


class ToolResultContentText(BaseModel):
    """text 类型 content 项."""
    type: str = "text"
    text: str


class ToolResultContentImage(BaseModel):
    """image 类型 content 项 (base64 编码)."""
    type: str = "image"
    source: dict[str, Any]  # { "type": "base64", "media_type": "image/png", "data": "..." }


class ToolResultContent(BaseModel):
    """ToolResult.content 数组中的单个元素（联合类型）."""
    type: str
    text: str | None = None
    source: dict[str, Any] | None = None


class ToolResultBlock(BaseModel):
    """
    对应 @anthropic-ai/sdk/resources/index.mjs 的 ToolResultBlockParam。
    即 LLM 看到的 tool_result 块。
    """
    type: str = "tool_result"
    tool_use_id: str
    content: list[ToolResultContent] | str
    is_error: bool | None = None


class ToolResultOutput(BaseModel, Generic[T]):
    """
    通用工具输出包装（对应 TypeScript ToolResult<Output>）.
    工具的 call() 方法返回此类型.
    """
    data: T
    new_messages: list[dict[str, Any]] | None = None
    context_modifier: Any = None
    mcp_meta: dict[str, Any] | None = None
```

### 3.2 `content` 字段的结构（text、image 等）

```python
# text content（最常用）
text_result = ToolResultBlock(
    tool_use_id="toolu_abc123",
    type="tool_result",
    content="Fetched content: Hello world",
)

# image content（截图等）
image_result = ToolResultBlock(
    tool_use_id="toolu_abc123",
    type="tool_result",
    content=[
        ToolResultContent(
            type="image",
            source={
                "type": "base64",
                "media_type": "image/png",
                "data": "iVBORw0KGgoAAAANS...",
            }
        )
    ],
)

# 复合 content（text + image）
composite_result = ToolResultBlock(
    tool_use_id="toolu_abc123",
    type="tool_result",
    content=[
        ToolResultContent(type="text", text="Screenshot captured:"),
        ToolResultContent(type="image", source={...}),
    ],
)
```

### 3.3 `is_error` 等标记字段

`is_error` 在 `ToolResultBlock` 级别设置，指示整个 tool_result 是否为错误：

```python
# 正常结果
normal_block = ToolResultBlock(
    tool_use_id="toolu_abc123",
    content="File written successfully",
    is_error=False,
)

# 错误结果（handler 返回结构化错误时）
error_block = ToolResultBlock(
    tool_use_id="toolu_abc123",
    content="Error: URL could not be parsed",
    is_error=True,
)
```

`map_tool_result_to_block_param` 方法负责将 `call()` 返回的 data 转换为 `ToolResultBlock`：

```python
def map_tool_result_to_block_param(self, content: Any, tool_use_id: str) -> dict[str, Any]:
    is_error = isinstance(content, dict) and content.get("error") is True
    return {
        "tool_use_id": tool_use_id,
        "type": "tool_result",
        "content": content if isinstance(content, str) else content.get("message", str(content)),
        "is_error": is_error or None,
    }
```

---

## 4. Tool 注册表 (`tools/registry.py`)

### 4.1 全局注册表的设计（dict + set）

```python
# tools/registry.py
from __future__ import annotations

import threading
from typing import Any, Callable

from .tool import ToolProtocol


class ToolRegistry:
    """
    全局单例工具注册表。
    对应 TypeScript src/tools.ts 中的 getAllBaseTools() / getTools() 逻辑。

    设计要点:
    - _tools: dict[str, ToolProtocol] — 按 name 索引，O(1) 查找
    - _tool_names: set[str] — 快速判断重复注册
    - _tags: dict[str, set[str]] — 按 tag 分组索引
    - 线程安全（读写锁）
    """

    def __init__(self):
        self._tools: dict[str, ToolProtocol] = {}
        self._tool_names: set[str] = set()
        self._tags: dict[str, set[str]] = {}
        self._lock = threading.RLock()

    # ── 注册 ────────────────────────────────────────────────

    def register(self, tool: ToolProtocol, tags: list[str] | None = None) -> None:
        """
        注册一个 Tool 实例。
        重复注册会抛出 ValueError。
        """
        with self._lock:
            if tool.name in self._tool_names:
                raise ValueError(
                    f"Tool '{tool.name}' is already registered. "
                    f"Use replace() to override an existing tool."
                )
            # 同时检查别名冲突
            for existing in self._tools.values():
                if hasattr(existing, 'aliases') and tool.name in existing.aliases:  # type: ignore
                    raise ValueError(
                        f"Tool name '{tool.name}' conflicts with alias of '{existing.name}'."
                    )
            self._tools[tool.name] = tool
            self._tool_names.add(tool.name)
            if tags:
                for tag in tags:
                    if tag not in self._tags:
                        self._tags[tag] = set()
                    self._tags[tag].add(tool.name)

    def replace(self, tool: ToolProtocol, tags: list[str] | None = None) -> None:
        """替换已存在的 Tool（用于 MCP 动态工具覆盖内置工具）."""
        with self._lock:
            old_tool = self._tools.get(tool.name)
            if old_tool is not None:
                self._tool_names.discard(old_tool.name)
            self._tools[tool.name] = tool
            self._tool_names.add(tool.name)
            if tags:
                for tag in tags:
                    if tag not in self._tags:
                        self._tags[tag] = set()
                    self._tags[tag].add(tool.name)

    # ── 查询 ────────────────────────────────────────────────

    def get(self, name: str) -> ToolProtocol | None:
        """按 name（或 alias）查找 Tool。"""
        with self._lock:
            tool = self._tools.get(name)
            if tool is None:
                # 检查 alias
                for t in self._tools.values():
                    if hasattr(t, 'aliases') and name in t.aliases:  # type: ignore
                        return t
            return tool

    def get_all(self) -> list[ToolProtocol]:
        """返回所有已注册工具的副本列表。"""
        with self._lock:
            return list(self._tools.values())

    def list_names(self) -> list[str]:
        """返回所有工具名称。"""
        with self._lock:
            return list(self._tool_names)

    def list_by_tag(self, tag: str) -> list[ToolProtocol]:
        """返回指定 tag 下的所有工具。"""
        with self._lock:
            names = self._tags.get(tag, set())
            return [self._tools[n] for n in names if n in self._tools]

    # ── 过滤 ────────────────────────────────────────────────

    def filter(
        self,
        predicate: Callable[[ToolProtocol], bool],
    ) -> list[ToolProtocol]:
        """返回满足 predicate 的工具列表。"""
        with self._lock:
            return [t for t in self._tools.values() if predicate(t)]

    def filter_by_names(self, names: list[str]) -> list[ToolProtocol]:
        """返回指定名称列表中的工具（保持顺序）。"""
        with self._lock:
            return [self._tools[n] for n in names if n in self._tools]

    # ── 权限过滤 ────────────────────────────────────────────

    def filter_by_permission_context(
        self,
        permission_context: dict[str, Any],
        deny_rules: dict[str, Any],
    ) -> list[ToolProtocol]:
        """
        根据权限上下文过滤工具。
        对应 TypeScript filterToolsByDenyRules() 的逻辑。
        """
        with self._lock:
            return [
                t for t in self._tools.values()
                if not self._is_denied(t, deny_rules)
            ]

    def _is_denied(
        self,
        tool: ToolProtocol,
        deny_rules: dict[str, Any],
    ) -> bool:
        """检查工具是否被 deny rule 匹配。"""
        tool_name = tool.name
        # 检查通配规则（如 mcp__server* 匹配所有该服务器工具）
        for rule_content in deny_rules.get("mcp", []):
            if tool_name.startswith(rule_content.rstrip("*")):
                return True
        # 检查精确规则
        return tool_name in deny_rules.get("exact", set())


# ─── 全局单例 ────────────────────────────────────────────

_registry = ToolRegistry()


def get_registry() -> ToolRegistry:
    return _registry


def register_tool(tool: ToolProtocol, tags: list[str] | None = None) -> None:
    get_registry().register(tool, tags)


def get_tool(name: str) -> ToolProtocol | None:
    return get_registry().get(name)


def list_tools() -> list[ToolProtocol]:
    return get_registry().get_all()


def list_tool_names() -> list[str]:
    return get_registry().list_names()
```

### 4.2 `tools_by_name` 索引的维护

```python
# 注册时
self._tools[tool.name] = tool          # 主索引，O(1)
self._tool_names.add(tool.name)        # 名称集合，O(1) 重复检查

# 查找时
tool = self._tools.get(name)           # O(1)

# 别名查找（线性扫描，但仅在精确查找失败时触发）
if tool is None:
    for t in self._tools.values():
        if name in getattr(t, 'aliases', []):
            return t
```

### 4.3 注册时的重复检查

```python
def register(self, tool: ToolProtocol, ...):
    if tool.name in self._tool_names:
        raise ValueError(f"Tool '{tool.name}' already registered.")
    # 同时检查别名冲突
    for existing in self._tools.values():
        if tool.name in getattr(existing, 'aliases', []):
            raise ValueError(
                f"Tool name '{tool.name}' conflicts with alias of '{existing.name}'."
            )
```

---

## 5. 内置 Tool 实现

### 5.1 `tools/web_fetch.py`

```python
# tools/web_fetch.py
from __future__ import annotations

import time
from typing import Any, Literal

import httpx
from pydantic import BaseModel, Field

from .factory import build_tool
from .tool import ToolResult, ToolUseContext


class WebFetchInput(BaseModel):
    url: str = Field(description="The URL to fetch content from")
    prompt: str = Field(description="The prompt to run on the fetched content")


class WebFetchOutput(BaseModel):
    bytes: int = Field(description="Size of the fetched content in bytes")
    code: int = Field(description="HTTP response code")
    codeText: str = Field(description="HTTP response code text")
    result: str = Field(description="Processed result from applying the prompt to the content")
    durationMs: int = Field(description="Time taken to fetch and process the content")
    url: str = Field(description="The URL that was fetched")


async def _fetch_url(url: str, abort_controller: Any = None) -> dict[str, Any]:
    """
    使用 httpx.AsyncClient 发送 HTTP GET 请求。
    对应 TypeScript WebFetchTool 的 getURLMarkdownContent() 逻辑.
    """
    timeout = httpx.Timeout(30.0, connect=10.0)
    async with httpx.AsyncClient(
        timeout=timeout,
        follow_redirects=True,
        headers={
            "User-Agent": "claude-code-tool/1.0 (WebFetchTool)",
            "Accept": "text/html, application/xhtml+xml, application/xml, text/markdown, */*",
        },
    ) as client:
        try:
            response = await client.get(url, follow_redirects=True)
            response.raise_for_status()
            return {
                "content": response.text,
                "bytes": len(response.content),
                "code": response.status_code,
                "codeText": response.reason_phrase,
                "contentType": response.headers.get("content-type", "text/plain"),
            }
        except httpx.TimeoutException as e:
            return {
                "error": True,
                "message": f"Request timed out after 30s: {str(e)}",
                "code": 0,
                "codeText": "Timeout",
            }
        except httpx.HTTPStatusError as e:
            return {
                "error": True,
                "message": f"HTTP {e.response.status_code}: {str(e)}",
                "code": e.response.status_code,
                "codeText": e.response.reason_phrase,
            }
        except Exception as e:
            return {
                "error": True,
                "message": f"Failed to fetch URL: {str(e)}",
                "code": 0,
                "codeText": "Error",
            }


async def _apply_prompt(content: str, prompt: str) -> str:
    """
    将 prompt 应用到抓取的 content 上。
    简化版本：直接返回 content + prompt 摘要。
    完整版本应调用 LLM 进行摘要。
    """
    preview = content[:2000] if len(content) > 2000 else content
    return f"[Content from URL]\n\nPrompt: {prompt}\n\nFirst 2000 chars:\n{preview}"


async def web_fetch_handler(
    args: dict[str, Any],
    context: ToolUseContext,
    can_use_tool: Any = None,
    parent_message: Any = None,
    on_progress: Any = None,
) -> ToolResult[dict[str, Any]]:
    """
    WebFetchTool 的核心 handler.
    对应 TypeScript WebFetchTool.call() 的逻辑.
    """
    start_ms = time.time()
    url: str = args["url"]
    prompt: str = args["prompt"]

    # 发送 HTTP 请求
    fetch_result = await _fetch_url(url, context.get("abortController"))

    # 处理错误：返回 error content 而非抛出异常
    if fetch_result.get("error"):
        duration_ms = int((time.time() - start_ms) * 1000)
        return ToolResult(
            data={
                "bytes": 0,
                "code": fetch_result["code"],
                "codeText": fetch_result["codeText"],
                "result": f"ERROR: {fetch_result['message']}",
                "durationMs": duration_ms,
                "url": url,
            }
        )

    # 应用 prompt 到 content
    processed = await _apply_prompt(fetch_result["content"], prompt)
    duration_ms = int((time.time() - start_ms) * 1000)

    return ToolResult(
        data={
            "bytes": fetch_result["bytes"],
            "code": fetch_result["code"],
            "codeText": fetch_result["codeText"],
            "result": processed,
            "durationMs": duration_ms,
            "url": url,
        }
    )


async def web_fetch_validate_input(
    args: dict[str, Any],
    context: ToolUseContext,
) -> dict[str, Any]:
    """验证 URL 格式."""
    url = args.get("url", "")
    try:
        parsed = httpx.URL(url)
        if parsed.scheme not in ("http", "https"):
            return {
                "result": False,
                "message": f'Invalid URL scheme "{parsed.scheme}". Only http and https are supported.',
                "errorCode": 1,
            }
        return {"result": True}
    except Exception:
        return {
            "result": False,
            "message": f'Error: Invalid URL "{url}". The URL provided could not be parsed.',
            "errorCode": 1,
        }


WebFetchTool = build_tool(
    name="web_fetch",
    input_schema=WebFetchInput,
    max_result_size_chars=100_000,
    should_defer=True,
    description=lambda input: f"Claude wants to fetch content from {input.get('url', 'this URL')}",
    validate_input=web_fetch_validate_input,
    is_read_only=lambda _: True,
    is_concurrency_safe=lambda _: True,
    to_auto_classifier_input=lambda input: (
        f"{input.get('url', '')}: {input.get('prompt', '')}"
    ),
    call=web_fetch_handler,
)
```

**超时处理：**

```python
timeout = httpx.Timeout(30.0, connect=10.0)  # 30s 总超时, 10s 连接超时
```

超时后 `httpx.TimeoutException` 被捕获，返回结构化错误结果而非抛出异常。

### 5.2 `tools/agent_tool.py`

```python
# tools/agent_tool.py
"""
AgentTool — 在子 agent 中执行任务。

对应 TypeScript src/tools/AgentTool/AgentTool.tsx 的逻辑。
核心调用 tasks/spawn.py 的 spawn_task()。
"""
from __future__ import annotations

from typing import Any, Literal

from pydantic import BaseModel, Field

from .factory import build_tool
from .tool import ToolResult, ToolUseContext


class AgentToolInput(BaseModel):
    description: str = Field(description="A short (3-5 word) description of the task")
    prompt: str = Field(description="The task for the agent to perform")
    subagent_type: str | None = Field(default=None, description="The type of specialized agent to use")
    model: Literal["sonnet", "opus", "haiku"] | None = Field(default=None, description="Model override")
    run_in_background: bool | None = Field(default=None, description="Run in background")
    # Multi-agent 参数（mode 决定权限模式）
    name: str | None = Field(default=None, description="Name for spawned agent")
    team_name: str | None = Field(default=None, description="Team name")
    mode: str | None = Field(default=None, description='Permission mode (e.g., "plan")')
    isolation: Literal["worktree"] | None = Field(default=None, description="Isolation mode")


class AgentToolOutput(BaseModel):
    status: Literal["completed", "async_launched"]
    agentId: str | None = None
    description: str | None = None
    result: str | None = None
    prompt: str | None = None


async def agent_call_handler(
    args: dict[str, Any],
    context: ToolUseContext,
    can_use_tool: Any = None,
    parent_message: Any = None,
    on_progress: Any = None,
) -> ToolResult[dict[str, Any]]:
    """
    AgentTool 的 call handler。
    对应 TypeScript AgentTool.call()，调用 spawn_task() 启动子 agent。
    """
    from your_framework import spawn_task  # 框架提供的 spawn_task 入口

    input_args = AgentToolInput.model_validate(args)

    spawn_kwargs: dict[str, Any] = {
        "prompt": input_args.prompt,
        "description": input_args.description,
        "agent_type": input_args.subagent_type or "general_purpose",
    }

    if input_args.model:
        spawn_kwargs["model"] = input_args.model

    if input_args.name:
        spawn_kwargs["name"] = input_args.name

    if input_args.team_name:
        spawn_kwargs["team_name"] = input_args.team_name

    # mode 参数传递（影响子 agent 的权限模式）
    if input_args.mode:
        spawn_kwargs["permission_mode"] = input_args.mode

    # isolation 参数（创建 git worktree 隔离）
    if input_args.isolation == "worktree":
        spawn_kwargs["isolation"] = "worktree"

    is_async = input_args.run_in_background or False

    try:
        if is_async:
            # 异步模式：立即返回 agentId
            agent_id = await spawn_task(
                **spawn_kwargs,
                background=True,
            )
            return ToolResult(
                data={
                    "status": "async_launched",
                    "agentId": agent_id,
                    "description": input_args.description,
                    "prompt": input_args.prompt,
                }
            )
        else:
            # 同步模式：等待结果
            result = await spawn_task(
                **spawn_kwargs,
                background=False,
            )
            return ToolResult(
                data={
                    "status": "completed",
                    "result": result,
                    "description": input_args.description,
                    "prompt": input_args.prompt,
                }
            )
    except Exception as e:
        return ToolResult(
            data={
                "status": "completed",
                "result": f"Error: {str(e)}",
            }
        )


AgentTool = build_tool(
    name="agent",
    input_schema=AgentToolInput,
    max_result_size_chars=100_000,
    description=lambda input: f"Run agent task: {input.get('description', '')}",
    is_concurrency_safe=lambda _: False,
    call=agent_call_handler,
)
```

**三种 mode（`local` / `remote` / `teammate`）的参数传递：**

```python
# TypeScript 中 permissionModeSchema 定义了 mode 的可能值
# Python 中用字符串字面量 + Literal 类型等效表达

spawn_kwargs: dict[str, Any] = {
    "permission_mode": mode,          # "plan" | "bypass" | "default"
    "isolation": isolation,            # "worktree" | None
    "team_name": team_name,            # 多 agent 团队协作
}
```

**结果包装：**

- 同步：`ToolResult(data={"status": "completed", "result": ...})`
- 异步：`ToolResult(data={"status": "async_launched", "agentId": ...})`

### 5.3 `tools/mcp_tool.py`

```python
# tools/mcp_tool.py
"""
MCPTool — 调用 MCP (Model Context Protocol) 服务器工具。

对应 TypeScript src/tools/MCPTool/MCPTool.ts 和 src/services/mcp/client.ts 的逻辑。
"""
from __future__ import annotations

from typing import Any

from pydantic import BaseModel, Field

from .factory import build_tool
from .tool import ToolResult, ToolUseContext


class MCPClientInterface:
    """
    MCP 客户端接口（对应 TypeScript MCPServerConnection）。
    框架应在运行时注入此接口的具体实现。
    """

    async def call_tool(
        self,
        tool_name: str,
        args: dict[str, Any],
        *,
        signal: Any = None,
        timeout_ms: int = 60000,
    ) -> dict[str, Any]:
        """
        调用 MCP 服务器上的工具。
        对应 TypeScript client.ts 中的 client.callTool() 调用。
        """
        ...


class MCPToolInput(BaseModel):
    """MCP 工具的输入是动态的（每个 MCP 服务器定义自己的 schema）。"""
    server: str = Field(description="MCP server name")
    tool: str = Field(description="Tool name on the MCP server")
    arguments: dict[str, Any] = Field(default_factory=dict, description="Tool arguments")


class MCPToolOutput(BaseModel):
    content: str | list[dict[str, Any]] = Field(description="Tool result content")
    is_error: bool | None = None


async def mcp_tool_handler(
    args: dict[str, Any],
    context: ToolUseContext,
    can_use_tool: Any = None,
    parent_message: Any = None,
    on_progress: Any = None,
) -> ToolResult[dict[str, Any]]:
    """
    MCPTool 的 call handler。
    关键步骤：
    1. 从 state store 获取 mcp_clients
    2. 找到目标 server 的 MCPClient
    3. 调用 client.call_tool()
    4. 结果转换为 ToolResult
    """
    input_args = MCPToolInput.model_validate(args)
    server_name = input_args.server
    tool_name = input_args.tool
    tool_args = input_args.arguments

    # Step 1: 从 AppState 获取 MCP clients
    app_state = context["getAppState"]()
    mcp_clients: list[MCPClientInterface] = app_state.get("mcp", {}).get("clients", [])

    # Step 2: 找到目标 server 的 client
    client: MCPClientInterface | None = None
    for c in mcp_clients:
        if hasattr(c, "server_name") and c.server_name == server_name:
            client = c
            break

    if client is None:
        return ToolResult(
            data={
                "content": f"Error: MCP server '{server_name}' not found",
                "is_error": True,
            }
        )

    # Step 3: 调用 MCPClient.call_tool()
    # 对应 TypeScript:
    #   const result = await Promise.race([
    #       client.callTool({ name: tool, arguments: args, _meta: meta }, ...),
    #       timeoutPromise,
    #   ])
    try:
        result = await client.call_tool(
            tool_name,
            tool_args,
            signal=context.get("abortController"),
            timeout_ms=60_000,
        )
    except Exception as e:
        # 错误时返回 error content 而非抛出异常
        return ToolResult(
            data={
                "content": f"MCP tool error: {str(e)}",
                "is_error": True,
            }
        )

    # Step 4: 结果转换（对应 TypeScript mapToolResultToToolResultBlockParam）
    is_error = result.get("isError", False)

    # 统一 content 格式
    content = result.get("content", [])
    if isinstance(content, list) and len(content) > 0:
        first = content[0]
        if isinstance(first, dict) and "text" in first:
            content_str = first["text"]
        else:
            content_str = str(content)
    else:
        content_str = str(content)

    return ToolResult(
        data={
            "content": content_str,
            "is_error": is_error or None,
        }
    )


MCPTool = build_tool(
    name="mcp",
    input_schema=MCPToolInput,
    max_result_size_chars=100_000,
    is_mcp=True,
    description="Execute a tool from an MCP server",
    is_concurrency_safe=lambda _: False,
    call=mcp_tool_handler,
)
```

**从 state store 获取 mcp_clients：**

```python
app_state = context["getAppState"]()
mcp_clients = app_state.get("mcp", {}).get("clients", [])
```

**调用 `MCPClient.call_tool()`：**

```python
result = await client.call_tool(
    tool_name,
    tool_args,
    signal=context["abortController"],
    timeout_ms=60_000,
)
```

对应 TypeScript `client.ts` 中的实现：

```typescript
// src/services/mcp/client.ts (line ~3091)
const result = await Promise.race([
  client.callTool(
    { name: tool, arguments: args, _meta: meta },
    CallToolResultSchema,
    { signal, timeout: timeoutMs, onprogress }
  ),
  timeoutPromise,
])
```

### 5.4 `tools/skill_tool.py`

```python
# tools/skill_tool.py
"""
SkillTool — 调用 skill（斜杠命令）。

对应 TypeScript src/tools/SkillTool/SkillTool.ts 的逻辑。
核心：从 skills/registry 获取 skill，调用 skill.get_prompt()。
"""
from __future__ import annotations

from typing import Any

from pydantic import BaseModel, Field

from .factory import build_tool
from .tool import ToolResult, ToolUseContext


class SkillToolInput(BaseModel):
    skill: str = Field(description='The skill name. E.g., "commit", "review-pr", or "pdf"')
    args: str | None = Field(default=None, description="Optional arguments for the skill")


class SkillToolOutput(BaseModel):
    success: bool
    commandName: str
    allowedTools: list[str] | None = None
    model: str | None = None
    status: str = "inline"  # "inline" | "forked"
    result: str | None = None
    agentId: str | None = None


async def get_all_commands(context: ToolUseContext) -> list[dict[str, Any]]:
    """
    获取所有可用命令（包括 MCP skills）。
    对应 TypeScript SkillTool.ts 的 getAllCommands() 逻辑。
    """
    from your_framework import get_commands, get_project_root

    # 获取本地命令
    local_commands = await get_commands(get_project_root())

    # 获取 MCP skills（从 AppState.mcp.commands 过滤 loadedFrom === 'mcp'）
    app_state = context["getAppState"]()
    mcp_commands = [
        cmd for cmd in app_state.get("mcp", {}).get("commands", [])
        if cmd.get("type") == "prompt" and cmd.get("loadedFrom") == "mcp"
    ]

    # 合并去重
    seen = set(cmd["name"] for cmd in local_commands)
    for cmd in mcp_commands:
        if cmd["name"] not in seen:
            local_commands.append(cmd)
            seen.add(cmd["name"])

    return local_commands


async def skill_tool_handler(
    args: dict[str, Any],
    context: ToolUseContext,
    can_use_tool: Any = None,
    parent_message: Any = None,
    on_progress: Any = None,
) -> ToolResult[dict[str, Any]]:
    """
    SkillTool 的 call handler。
    步骤：
    1. 获取 skill 名称和参数
    2. 从 registry 查找 skill
    3. 调用 skill.get_prompt() 获取展开后的消息
    4. 通过 contextModifier 注入 newMessages
    """
    from your_framework import find_command, process_prompt_slash_command

    input_args = SkillToolInput.model_validate(args)
    skill_name = input_args.skill.strip().lstrip("/")
    skill_args = input_args.args or ""

    # 获取所有可用命令
    commands = await get_all_commands(context)

    # 查找 skill
    command = find_command(skill_name, commands)
    if command is None:
        return ToolResult(
            data={
                "success": False,
                "commandName": skill_name,
                "result": f"Unknown skill: {skill_name}",
            }
        )

    # 检查是否需要 fork 执行（context === 'fork'）
    if command.get("context") == "fork":
        return await _execute_forked_skill(command, skill_name, skill_args, context)

    # 处理 inline skill（对应 processPromptSlashCommand）
    processed = await process_prompt_slash_command(
        command,
        skill_args,
        commands,
        context,
    )

    if not processed.should_query:
        return ToolResult(
            data={
                "success": False,
                "commandName": skill_name,
                "result": "Command processing failed",
            }
        )

    allowed_tools = processed.allowed_tools or []
    model = processed.model

    # 通过 newMessages 注入展开后的消息
    tool_use_id = (
        parent_message.get("message", {}).get("id", "")
        if parent_message else ""
    )
    new_messages = [
        {**msg, "tool_use_id": tool_use_id}
        for msg in processed.messages
        if msg.get("type") in ("user", "system", "attachment")
    ]

    def context_modifier(ctx: ToolUseContext) -> ToolUseContext:
        """修改 context（注入 allowedTools、model 等）。"""
        modified = dict(ctx)

        if allowed_tools:
            app_state = ctx["getAppState"]()
            new_state = dict(app_state)
            new_state["toolPermissionContext"] = dict(
                app_state.get("toolPermissionContext", {})
            )
            new_state["toolPermissionContext"]["alwaysAllowRules"] = {
                **new_state["toolPermissionContext"].get("alwaysAllowRules", {}),
                "command": list(set([
                    *new_state["toolPermissionContext"]
                    .get("alwaysAllowRules", {})
                    .get("command", []),
                    *allowed_tools,
                ])),
            }

            def new_get_app_state():
                return new_state

            modified["getAppState"] = new_get_app_state

        if model:
            modified["options"] = {
                **ctx.get("options", {}),
                "mainLoopModel": model,
            }

        return modified  # type: ignore

    return ToolResult(
        data={
            "success": True,
            "commandName": skill_name,
            "allowedTools": allowed_tools or None,
            "model": model,
            "status": "inline",
        },
        new_messages=new_messages,
        context_modifier=context_modifier,
    )


async def _execute_forked_skill(
    command: dict[str, Any],
    command_name: str,
    args: str,
    context: ToolUseContext,
) -> ToolResult[dict[str, Any]]:
    """
    Fork 模式：在独立子 agent 中执行 skill。
    对应 TypeScript executeForkedSkill()。
    """
    from your_framework import run_agent, create_agent_id

    agent_id = create_agent_id()

    agent_messages = []
    async for message in run_agent(
        agent_definition={
            "agentType": command.get("agent", {}).get("type", "general_purpose"),
            "promptMessages": [],
        },
        tool_use_context=context,
        is_async=False,
    ):
        agent_messages.append(message)

    result_text = _extract_result_text(agent_messages)

    return ToolResult(
        data={
            "success": True,
            "commandName": command_name,
            "status": "forked",
            "agentId": agent_id,
            "result": result_text,
        }
    )


SkillTool = build_tool(
    name="skill",
    input_schema=SkillToolInput,
    max_result_size_chars=100_000,
    description=lambda input: f"Execute skill: {input.get('skill', '')}",
    is_concurrency_safe=lambda _: False,
    call=skill_tool_handler,
)
```

**`contextModifier` 注入 newMessages 的逻辑：**

```python
# SkillTool.call() 返回 ToolResult 时携带 new_messages
return ToolResult(
    data={...},
    new_messages=[user_message, attachment_message, ...],
    context_modifier=lambda ctx: {
        **ctx,
        "getAppState": lambda: {
            ...ctx["getAppState"](),
            "toolPermissionContext": {
                ...,
                "alwaysAllowRules": { ..., "command": allowed_tools },
            }
        }
    }
)
```

`new_messages` 由调用方（如 `query.ts` 主循环）注入到消息列表中，后续 LLM turn 会看到展开后的 skill 内容。

---

## 6. Tool 调用完整流程

### 6.1 序列图

```
LLM (Claude API)
    │
    │ tool_use block { name: "web_fetch", args: { url, prompt } }
    ▼
───────────────────────────────────────────────────────────────────
                        Query Engine (主循环)
───────────────────────────────────────────────────────────────────
    │
    │ 1. 解析 tool_use 请求
    ▼
    │ 2. 从 Registry 查找工具: get_tool("web_fetch")
    │    registry.get("web_fetch") → WebFetchTool
    │
    ▼
    │ 3. validateInput(args, context) — 同步参数预校验（如 URL 格式）
    │    ├─ 成功 → { result: true }
    │    └─ 失败 → { result: false, message, errorCode }
    │
    ▼
    │ 4. checkPermissions(args, context) — 权限检查
    │    ├─ allow → 继续
    │    ├─ deny → 返回 PermissionDenied 错误
    │    └─ ask → 弹出用户确认 UI
    │
    ▼
    │ 5. build_tool 内部的 _validate_args(args)
    │    └─ Pydantic ValidationError → ToolDefError → 返回结构化错误
    │
    ▼
    │ 6. 调用 Tool.call(args, context, can_use_tool, parent_message, on_progress)
    │    └─ web_fetch_handler(args, context)
    │         ├─ httpx.AsyncClient.get(url)
    │         ├─ 错误 → return ToolResult(data={ is_error: True, ... })
    │         └─ 成功 → return ToolResult(data={ result, bytes, code, ... })
    │
    ▼
    │ 7. mapToolResultToBlockParam(content, toolUseId)
    │    └─ { tool_use_id, type: "tool_result", content: "...", is_error: true/false }
    │
    ▼
    │ 8. 将 ToolResultBlock 返回给 API 消费
    │
    ▼
LLM (继续处理 tool_result 块，生成下一个 turn)
```

### 6.2 步骤分解

| 步骤 | 组件 | 操作 |
|------|------|------|
| 1 | `query.ts` | 接收 API 的 `tool_use` 事件 |
| 2 | `registry.get(name)` | 从全局注册表查找 Tool 实例 |
| 3 | `tool.validateInput(args, ctx)` | 同步参数预校验（如 URL 格式） |
| 4 | `tool.checkPermissions(args, ctx)` | 权限规则匹配（allow/deny/ask） |
| 5 | `build_tool._validate_args()` | Pydantic schema 强制验证 |
| 6 | `tool.call(args, ctx, ...)` | 执行业务逻辑 |
| 7 | `tool.mapToolResultToBlockParam()` | 转换为 SDK 标准块格式 |
| 8 | 返回 `ToolResultBlock` | 注入消息流，LLM 继续 |

**错误处理流程：**

```
Schema 验证失败 (Pydantic ValidationError)
    ↓
build_tool._validate_args() 捕获
    ↓
返回 ToolResult(data={ "error": True, "message": "...", "errorCode": 1 })
    ↓
mapToolResultToBlockParam 设置 is_error=True
    ↓
LLM 看到 tool_result with is_error=true
```

```
Handler 内部异常 (httpx.TimeoutException)
    ↓
handler 捕获异常
    ↓
返回 ToolResult(data={ "is_error": True, "content": "Timeout after 30s" })
    ↓
is_error=True 传递到 ToolResultBlock
```

---

## 7. 扩展新 Tool 的模式

### 7.1 按照 `build_tool` 模式添加新工具的步骤

**Step 1: 定义输入 Schema（Pydantic Model）**

```python
# tools/my_tool.py
from pydantic import BaseModel, Field
from typing import Literal

class MyToolInput(BaseModel):
    action: Literal["create", "read", "delete"] = Field(description="Action to perform")
    path: str = Field(description="Target file path")
    content: str | None = Field(default=None, description="Content for create/write actions")
```

**Step 2: 定义输出 Schema（可选）**

```python
class MyToolOutput(BaseModel):
    success: bool
    message: str
    data: dict[str, Any] | None = None
```

**Step 3: 实现 handler 函数**

```python
async def my_tool_handler(
    args: dict[str, Any],
    context: ToolUseContext,
    can_use_tool: Any = None,
    parent_message: Any = None,
    on_progress: Any = None,
) -> ToolResult[dict[str, Any]]:
    validated = MyToolInput.model_validate(args)

    if validated.action == "read":
        return ToolResult(data={"success": True, "message": "Read ok", "data": {...}})
    elif validated.action == "create":
        return ToolResult(data={"success": True, "message": "Created", "data": {...}})
    else:
        return ToolResult(data={"success": False, "message": "Unknown action"})
```

**Step 4: 实现 validateInput（可选，推荐）**

```python
async def my_tool_validate(
    args: dict[str, Any],
    context: ToolUseContext,
) -> dict[str, Any]:
    if not args.get("path"):
        return {"result": False, "message": "path is required", "errorCode": 1}
    return {"result": True}
```

**Step 5: 调用 build_tool 创建 Tool 实例**

```python
MyTool = build_tool(
    name="my_tool",
    input_schema=MyToolInput,
    description=lambda input: f"MyTool: {input.get('action')} on {input.get('path')}",
    validate_input=my_tool_validate,
    is_read_only=lambda input: input.get("action") == "read",
    is_destructive=lambda input: input.get("action") == "delete",
    is_concurrency_safe=lambda input: input.get("action") == "read",
    to_auto_classifier_input=lambda input: f"{input.get('action')} {input.get('path')}",
    call=my_tool_handler,
    max_result_size_chars=50_000,
)
```

**Step 6: 注册到全局注册表**

```python
from tools.registry import register_tool

register_tool(MyTool, tags=["file_operations"])
```

### 7.2 注意事项

1. **错误不抛出异常**：所有错误通过 `ToolResult(data={is_error: True, ...})` 返回
   - 原因：抛出异常会中断主循环，导致会话崩溃
   - 结构化错误让 LLM 能看到错误信息并尝试恢复

2. **Pydantic ValidationError 处理**：在 `build_tool` 层面统一捕获，不要在 handler 中逐个 try/catch

3. **并发安全**：`isConcurrencySafe` 返回 `False` 时，Query Engine 不会并行调用此工具

4. **权限检查**：敏感工具（文件操作、网络请求）必须实现 `checkPermissions`，返回 `allow/deny/ask`

5. **`newMessages` 注入**：工具需要向会话添加消息（如 SkillTool 展开 slash 命令）时，通过 `ToolResult.new_messages` 返回，由调用方负责合并

6. **`contextModifier`**：需要修改全局状态（如 allowedTools、model）时，通过 `ToolResult.context_modifier` 函数返回，由调用方在下一个 turn 应用

7. **`maxResultSizeChars`**：大输出工具（文件读取、搜索结果）应设置此值，超过阈值时结果持久化到磁盘

8. **别名支持**：如果工具需要兼容旧名称，通过 `aliases: ["old_name"]` 参数注册

9. **`alwaysLoad` / `shouldDefer`**：需要 ToolSearch 功能的工具设置 `shouldDefer=True`；模型必须在首轮看到的工具设置 `alwaysLoad=True`

---

## 附录：TypeScript → Python 类型对照表

| TypeScript (`src/Tool.ts`) | Python 等价 |
|---|---|
| `type Tool<Input, Output, P>` | `class ToolProtocol(Protocol)` |
| `ToolDef<Input, Output, P>` | `dict[str, Any]`（传入 `build_tool()` 的参数字典） |
| `ToolUseContext` | `@TypedDict class ToolUseContext` |
| `ToolResult<T>` | `@dataclass class ToolResult(Generic[T])` |
| `ValidationResult` | `TypedDict("ValidationResult", result=True)` / `ValidationResultError` |
| `ToolPermissionContext` | `@TypedDict class ToolPermissionContext` |
| `buildTool<D>(def: D)` | `build_tool(**def_dict)` |
| `TOOL_DEFAULTS` | `TOOL_DEFAULTS: ToolDefaults` dataclass |
| `toolMatchesName(tool, name)` | `get_tool(name)` + alias 遍历 |
| `Tools = readonly Tool[]` | `list[ToolProtocol]`（注册表返回 `list[ToolProtocol]`） |

## 附录：核心文件索引

| 功能 | TypeScript 文件 | Python 文件 |
|---|---|---|
| Tool 接口定义 | `src/Tool.ts` (line 362-695) | `tools/tool.py` |
| buildTool 工厂 | `src/Tool.ts` (line 783-792) | `tools/factory.py` |
| 工具注册表 | `src/tools.ts` | `tools/registry.py` |
| WebFetchTool | `src/tools/WebFetchTool/WebFetchTool.ts` | `tools/web_fetch.py` |
| AgentTool | `src/tools/AgentTool/AgentTool.tsx` | `tools/agent_tool.py` |
| MCPTool | `src/tools/MCPTool/MCPTool.ts` | `tools/mcp_tool.py` |
| SkillTool | `src/tools/SkillTool/SkillTool.ts` | `tools/skill_tool.py` |
| MCP call_tool | `src/services/mcp/client.ts` (line ~3091) | `MCPClientInterface.call_tool()` |
