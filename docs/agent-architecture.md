# 最小可用 Agent 架构（Python 版）

## 概述

目标：**一个人能在 1-2 周内启动一个可工作的 agent 循环**。

核心组件：Agent Loop、Tools、Skills、MCP、State、Subagents。省略：CLI 界面层、桥接协议、voice、analytics、compactor、文件操作、Shell、slash commands。

技术栈：Python 3.11+，类型提示（Pydantic），异步编程（asyncio），**claude-agent-sdk**（与本项目相同的 Agent SDK 的 Python 版本）。

## 目录结构

```
src/
├── agent/                      # Agent 核心循环
│   ├── loop.py                 # 主 while-true 循环，接收消息 → 调用 LLM → 处理 tool_use
│   ├── query_engine.py         # SDK query 生命周期：stream 管理、token 计算、停止信号
│   └── types.py                # Agent 相关类型定义（Pydantic 模型）
│
├── tools/                      # Tools 系统
│   ├── tool.py                 # Tool 接口定义 + build_tool 工厂函数
│   ├── registry.py             # Tool 注册表，全局工具收集 + filtering
│   ├── web_fetch.py            # HTTP 请求（唯一的外部 I/O）
│   ├── agent_tool.py           # 启动子 Agent（调用 tasks/）
│   ├── mcp_tool.py             # 调用 MCP server 工具
│   └── skill_tool.py           # 调用 Skill
│
│   # 注意：本架构不包含文件操作和 Shell 操作
│   # 如需文件操作，通过 MCP server（如 FS MCP）或者 AgentTool 调度子 Agent 来完成
│
├── tasks/                      # Subagent / 任务系统
│   ├── types.py               # Task 定义、TaskResult、TaskProgress 类型
│   ├── task.py               # Task 基类或工厂
│   ├── local_agent.py         # 本地子 Agent（同进程内运行）
│   ├── remote_agent.py         # 远程 Agent（通过 HTTP 调度）
│   ├── teammate.py             # 进程内队友（共享状态，协作完成子任务）
│   └── spawn.py               # Task 统一入口：spawn_task(type, config)
│
├── skills/                     # Skills 系统
│   ├── skill.py                # Skill 接口定义（Pydantic 模型）
│   ├── registry.py             # Skill 全局注册表，供 agent 触发时查找
│   ├── loader.py               # 从磁盘目录动态加载 skill（遵循 Anthropic 规范）
│   │                           #   - 格式：skill-name/SKILL.md
│   │                           #   - frontmatter 字段：name, description, user-invocable,
│   │                           #     allowed-tools, argument-hint, arguments, when-to-use,
│   │                           #     model, disable-model-invocation, context, agent,
│   │                           #     effort, paths, hooks, shell
│   │                           #   - 内置变量替换：${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
│   │                           #   - 条件触发：paths frontmatter（匹配文件路径时激活）
│   │                           #   - 支持 inline shell 执行（!`...`，安全限制同 MCP）
│   │                           #   - 加载来源：bundled, user, project, managed
│   └── bundled/                # 内置 skills（遵循同一 SKILL.md 格式）
│       ├── simplify/SKILL.md
│       ├── verify/SKILL.md
│       ├── brainstorming/SKILL.md
│       └── ...                  # 按需添加更多内置 skill
│
├── mcp/                        # MCP (Model Context Protocol) Client
│   ├── client.py               # MCP HTTP/SSE client 实现
│   ├── types.py                # ServerConfig、Transport、Resource 等类型（Pydantic）
│   └── normalize.py            # 工具名称/参数归一化
│
├── state/                      # State 管理
│   ├── store.py                # 全局状态 Store，支持 subscribe（类似 Zustand）
│   └── selectors.py            # 状态选择器（派生数据）
│
├── context/                    # 上下文管理
│   ├── __init__.py
│   └── messages.py             # 用户输入 → 可发送给 LLM 的消息格式
│
├── utils/                      # 工具函数
│   ├── logger.py               # 日志输出
│   ├── retry.py                # 重试逻辑
│   └── pydantic_utils.py       # Pydantic schema 工具函数
│
├── config.py                   # 全局配置（API Key、MCP servers 等，统一流向各模块）
│
└── main.py                    # 入口文件（启动 headless 模式）
```

## 核心模块职责

| 模块 | 文件 | 职责 | 关键接口 |
|------|------|------|----------|
| **Agent Loop** | `agent/loop.py` | while-true 消息循环，调用 LLM，处理 tool_use 块 | `run_agent_loop(messages: list[Message]) -> AsyncGenerator[Message]` |
| **Message Types** | `agent/types.py` | HumanMessage、AssistantMessage、ToolResult 等 Pydantic 模型 | `HumanMessage(content=...)`, `AssistantMessage(content=..., tool_use=[...])` |
| **QueryEngine** | `agent/query_engine.py` | 管理流式响应、token 上报、停止信号 | `QueryEngine.create(config)` → `.run()` → stream |
| **Tool** | `tools/tool.py` | 工具接口：name/description/input_schema/handler | `Tool(name, description, schema, handler)` |
| **Tool Registry** | `tools/registry.py` | 全局工具注册，过滤，查找 | `register_tool()`, `get_tool()`, `list_tools()` |
| **Task / Subagent** | `tasks/task.py` | 子 Agent 调度：local/remote/teammate 三种模式 | `spawn_task(type, config)` → `Task.run()` |
| **Skill** | `skills/skill.py` | Skill in-memory 表示：frontmatter 解析后的元数据 + prompt getter | `Skill(name, description, get_prompt, allowed_tools, ...)` |
| **MCP Client** | `mcp/client.py` | 连接 MCP server，调用其工具，订阅 resources | `MCPClient.connect(config)` → `call_tool()` / `list_resources()` |
| **AppState** | `state/store.py` | 全局可变状态，订阅变更通知 | `create_store(initial_state)` → `{ get, set, subscribe }` |
| **Context** | `context/messages.py` | 用户输入 → 构造可发送给 LLM 的消息列表 | `build_context(user_input, state)` → `list[Message]` |

## 数据流

```
用户输入
  → context/ (构造消息)
    → agent/loop.py (LLM 调用)
      → QueryEngine (流式响应)
        ├── tool_use 块
        │   → tools/registry.py (查找工具)
        │     → Tool.handler() 执行
        │       ├── mcp/client.py (MCP 工具) 或 本地工具
        │       └── AgentTool → tasks/ (Subagent 调度)
        └── skill 触发 (以特定 keyword 开头)
            → skills/ (动态加载或内置)
      → 最终响应 → 用户
```

## Subagent 三种模式

| 类型 | 运行位置 | 通信方式 | 适用场景 |
|------|----------|----------|----------|
| `LocalAgentTask` | 同进程内 | `await` 协程调用，直接返回 | 快速、共享内存、无网络开销 |
| `RemoteAgentTask` | 远程 HTTP Endpoint | HTTP/JSON-RPC 回调 | 隔离进程、跨服务、并行构建 |
| `InProcessTeammateTask` | 同进程 | 共享 state store + 消息队列 | 需要实时协作、状态同步 |

## Tool 接口示例

```python
# tools/tool.py
from dataclasses import dataclass
from typing import Any, Awaitable, Callable, TypeVar
from pydantic import BaseModel

T = TypeVar("T", bound=BaseModel)

class ToolResult(BaseModel):
    content: list[dict[str, Any]]


@dataclass
class Tool:
    name: str
    description: str
    input_schema: type[BaseModel]
    handler: Callable[[BaseModel, "ToolUseContext"], Awaitable[ToolResult]]


ToolUseContext = dict[str, Any]  # 运行时上下文（消息历史、状态快照等）


def build_tool(
    name: str,
    description: str,
    schema: type[BaseModel],
    handler: Callable[[BaseModel, ToolUseContext], Awaitable[ToolResult]],
) -> Tool:
    return Tool(name=name, description=description, input_schema=schema, handler=handler)


# tools/agent_tool.py — 调度子 Agent
from typing import Literal
from pydantic import BaseModel


class AgentInput(BaseModel):
    task: str
    mode: Literal["local", "remote", "teammate"] = "local"
    model: str | None = None


async def agent_handler(input: AgentInput, context: ToolUseContext) -> ToolResult:
    from tasks.spawn import spawn_task

    task = spawn_task(input.mode, {
        "task": input.task,
        "model": input.model,
        "parent_context": context,
    })
    result = await task.run()
    return ToolResult(content=[{"type": "text", "text": result}])


AgentTool = build_tool(
    name="Agent",
    description="Spawn a sub-agent to complete a task in parallel",
    schema=AgentInput,
    handler=agent_handler,
)


# tools/web_fetch.py — 唯一的外部 I/O 通道
import httpx
from pydantic import BaseModel


class WebFetchInput(BaseModel):
    url: str
    prompt: str | None = None


async def web_fetch_handler(input: WebFetchInput, context: ToolUseContext) -> ToolResult:
    async with httpx.AsyncClient() as client:
        response = await client.get(input.url, timeout=30.0)
        text = response.text
    return ToolResult(content=[{"type": "text", "text": text}])


WebFetchTool = build_tool(
    name="WebFetch",
    description="Fetch content from a URL",
    schema=WebFetchInput,
    handler=web_fetch_handler,
)
```

## Skill 接口示例

```python
# skills/skill.py — frontmatter 解析后的 in-memory 表示
from pydantic import BaseModel
from typing import Any


class Skill(BaseModel):
    name: str
    description: str
    user_invocable: bool = True
    allowed_tools: list[str] = []
    argument_hint: str | None = None
    arguments: list[str] = []
    when_to_use: str | None = None
    model: str | None = None
    disable_model_invocation: bool = False
    context: Literal["inline", "fork"] | None = None
    agent: str | None = None
    effort: Literal["low", "medium", "high"] | None = None
    paths: list[str] | None = None   # 条件触发：匹配文件路径时激活
    hooks: dict | None = None
    shell: dict | None = None
    markdown_content: str               # 原始 markdown 正文（包含 frontmatter 后的内容）
    base_dir: str | None = None         # 所在目录，替换 ${CLAUDE_SKILL_DIR}
    source: Literal["bundled", "user", "project", "managed"] = "bundled"

    async def get_prompt(
        self, args: str, tool_use_context: dict[str, Any]
    ) -> list[dict[str, Any]]:
        """
        返回发送给 LLM 的 prompt 内容块。
        执行变量替换：${CLAUDE_SKILL_DIR}、${CLAUDE_SESSION_ID}、${arg1} 等。
        支持 inline shell 执行（!`...`），安全限制同 MCP 规范。
        """
        content = self.markdown_content

        if self.base_dir:
            content = content.replace("${CLAUDE_SKILL_DIR}", self.base_dir)

        content = content.replace("${CLAUDE_SESSION_ID}", "session-id")

        # 参数替换 ${arg1}, ${arg2}, ...
        for i, arg_name in enumerate(self.arguments):
            content = content.replace(f"${{{arg_name}}}", args.split()[i] if i < len(args.split()) else "")

        return [{"type": "text", "text": content}]


# skills/loader.py — 遵循 Anthropic SKILL.md 规范
# 1. 扫描 skills/ 目录（bundled/user/project/managed）下的 skill-name/SKILL.md
# 2. 解析 frontmatter（name, description, user-invocable, allowed-tools, 等）
# 3. 实例化 Skill，markdown_content = frontmatter 后的纯内容
# 4. 注册到 skills/registry.py
#
# 目录格式：
#   skills/bundled/simplify/SKILL.md
#   skills/bundled/verify/SKILL.md
#   ~/.my-agent/skills/brainstorming/SKILL.md
#   .claude/skills/triage/SKILL.md
```
```

## MCP Client 接口示例

```python
# mcp/client.py
from typing import Any, Literal
from pydantic import BaseModel


class MCPServerConfig(BaseModel):
    url: str
    transport: Literal["http", "sse"] = "http"
    headers: dict[str, str] = {}


class Resource(BaseModel):
    uri: str
    name: str
    mime_type: str | None = None


class MCPClient:
    async def connect(self, config: MCPServerConfig) -> None: ...

    async def call_tool(self, name: str, args: dict[str, Any]) -> dict[str, Any]: ...

    async def list_resources(self) -> list[Resource]: ...

    def subscribe_resource(self, uri: str, callback: callable) -> None: ...

    async def disconnect(self) -> None: ...
```

## AppState Store 示例

```python
# state/store.py
from typing import TypeVar, Generic, Callable, Awaitable
from dataclasses import dataclass, field

T = TypeVar("T")


@dataclass
class Store(Generic[T]):
    _state: T = field(default_factory=dict)
    _subscribers: set[Callable[[T], Awaitable[None] | None]] = field(default_factory=set)

    def get(self) -> T:
        return self._state

    def set(self, patch: dict) -> None:
        self._state = {**self._state, **patch}
        for fn in self._subscribers:
            fn(self._state)

    def subscribe(self, fn: Callable[[T], Awaitable[None] | None]) -> Callable[[], None]:
        self._subscribers.add(fn)
        return lambda: self._subscribers.discard(fn)


def create_store(initial: T) -> Store[T]:
    return Store(_state=initial)


## Message 类型示例

```python
# agent/types.py
from pydantic import BaseModel
from typing import Any


class HumanMessage(BaseModel):
    role: Literal["user"] = "user"
    content: str


class AssistantMessage(BaseModel):
    role: Literal["assistant"] = "assistant"
    content: str
    tool_use: list[dict[str, Any]] = []   # LLM 决定调用工具时填充
    stop_reason: str | None = None


class ToolResultMessage(BaseModel):
    role: Literal["user"] = "user"        # tool result 视为 user 角色回传
    content: str
    tool_use_id: str                       # 对应 tool_use 块的 id


# 联合类型用于消息历史
Message = HumanMessage | AssistantMessage | ToolResultMessage
```


## main.py 入口示例

```python
# main.py
import asyncio
from agent.loop import run_agent_loop
from context.messages import build_context
from state.store import create_store
from tools.registry import list_tools
from skills.loader import load_all_skills
from mcp.client import MCPClient
from config import get_config


async def main() -> None:
    config = get_config()

    # 初始化全局状态
    state = create_store({
        "messages": [],
        "mcp_clients": {},
    })

    # 初始化 MCP servers（从 config 读取）
    mcp_clients: dict[str, MCPClient] = {}
    for server_cfg in config.mcp_servers:
        client = MCPClient()
        await client.connect(server_cfg)
        mcp_clients[server_cfg.name] = client
    state.set({"mcp_clients": mcp_clients})

    # 加载 skills
    skills = await load_all_skills(config.skills_dir)

    # 注册 tools（包含 WebFetchTool, AgentTool, MCPTool, SkillTool）
    tools = list_tools()

    # 构建初始上下文
    messages = build_context([], state.get())

    # 主循环
    async for response in run_agent_loop(messages, state, tools, skills):
        print(response.content)


if __name__ == "__main__":
    asyncio.run(main())
```


## 扩展路径

## 扩展路径

```
当前最小版               未来可扩展
─────────────────       ───────────────────────
tools/web_fetch.py     tools/ → 更多业务工具（按需添加）
                       文件/Shell → 通过 MCP server 或 AgentTool 子 Agent 完成
skills/bundled/         skills/ → 动态目录加载 + 用户自定义
mcp/client.py (HTTP)     mcp/ → SSE, WebSocket, stdio transport
state/store.py           state/ → 历史消息、session 持久化
无多 Agent               tasks/ → LocalAgent, RemoteAgent, InProcessTeammate
```

## 构建命令（参考）

```bash
# 安装依赖
pip install -r requirements.txt

# 依赖（最小集）
# claude-agent-sdk>=0.1.59  # Anthropic Python Agent SDK
# pydantic>=2.0
# httpx>=0.25

# 类型检查
mypy src/

# 运行
python -m src.main
```

## 项目初始化文件

```
src/
├── __init__.py
├── config.py              # 全局配置（API Key、MCP servers）
├── main.py                # 入口
├── agent/__init__.py
├── tools/__init__.py
├── tasks/__init__.py
├── skills/__init__.py
├── skills/registry.py     # Skill 全局注册表
├── skills/bundled/         # 内置 skills（SKILL.md 格式）
├── mcp/__init__.py
├── state/__init__.py
├── context/__init__.py
└── utils/__init__.py
```
