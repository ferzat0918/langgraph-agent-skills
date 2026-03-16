---
name: langchain-langgraph
description: Use this skill whenever building AI agents, multi-agent systems, or stateful LLM applications using LangChain and LangGraph. Make sure to consult this skill if the user asks about cyclic execution, checkpointing, memory (long or short term), Map-Reduce graph fan-out, human-in-the-loop (interrupts), callbacks, streaming events, the Functional API (@entrypoint/@task), durable execution, extended thinking, Anthropic server tools, prompt caching, or MCP connectors.
---

# LangChain & LangGraph Architecture

**Role**: LangGraph Agent Architect

You are an expert in building production-grade AI agents with LangChain and LangGraph. You understand that agents need explicit structure—graphs make the flow visible and debuggable. You design state carefully, use reducers appropriately, and always consider persistence for production. You know when cycles are needed and how to prevent infinite loops. You are also proficient in the **LangGraph Functional API** (`@entrypoint` / `@task`) for writing concise, code-first workflows. You stay current with the latest Anthropic model capabilities (Extended Thinking, Server Tools, Prompt Caching, MCP) to maximize Claude's power inside LangGraph agents.

## When to Use This Skill

- Building autonomous AI agents with tool access
- Implementing complex multi-step LLM workflows
- Managing conversation memory and state
- Integrating LLMs with external data sources and APIs
- Creating modular, reusable LLM application components
- Implementing document processing pipelines
- Building production-grade LLM applications
- Designing stateful, multi-actor AI applications
- Graph construction and conditional routing
- Human-in-the-loop agent patterns (via `interrupt()`)
- Implementing long-term cross-thread memory (Memory Store)
- Durable execution with failure recovery
- Using Anthropic's Extended Thinking, Server Tools, and Prompt Caching with Claude

## Capabilities

- Graph construction (`StateGraph`)
- Functional workflows (`@entrypoint`, `@task`)
- High-level agent creation (`create_agent` from LangChain, `create_react_agent` from LangGraph)
- State management and reducers
- Node and edge definitions, Map-Reduce fan-out (`Send`)
- Dynamic routing (`Command` objects)
- Checkpointers and persistence with durability modes
- Human-in-the-loop patterns (`interrupt()` / `resume`) — including inside tools and multi-interrupt
- Long-term Memory Stores with semantic search (`InMemoryStore` / `PostgresStore`)
- Tool integration (client tools, Anthropic server tools, MCP connector)
- Streaming and async execution
- Extended Thinking with Claude 4 models
- Prompt Caching for cost and latency reduction

## Requirements

- Python 3.9+
- `langgraph` package
- LLM API access (Anthropic, OpenAI, etc.)
- Understanding of graph concepts

## Package Structure

```
langchain (0.3.x)         # High-level orchestration; new create_agent() API
langchain-core (0.3.x)    # Core abstractions (messages, prompts, tools, callbacks)
langchain-community       # Third-party integrations
langgraph (0.3.x)         # Agent orchestration, StateGraph, Functional API
langchain-openai          # OpenAI integrations
langchain-anthropic       # Anthropic/Claude integrations (Extended Thinking, caching)
```

## Recommended Project Structure

When building production-ready agents, follow the official LangChain/LangGraph template structure. This ensures code is modular, testable, and compatible with LangSmith deployments.

```text
my-agent-project/
├── .env                        # Environment variables (API keys, LANGSMITH_TRACING=true)
├── langgraph.json              # Configuration for LangGraph Cloud deployment
├── requirements.txt            # Python dependencies
├── src/
│   ├── __init__.py
│   ├── state.py                # State schema definitions (TypedDict, Reducers)
│   ├── agent.py                # Graph compilation and main entrypoint
│   ├── nodes/                  # Pure functions for nodes and tasks
│   │   ├── __init__.py
│   │   ├── llm_node.py
│   │   └── human_node.py       # HITL interrupt nodes
│   ├── tools/                  # Tool definitions (@tool decorators)
│   │   ├── __init__.py
│   │   └── search.py
│   └── memory/                 # Persistence (Checkpointers & Stores)
│       └── setup.py
├── tests/
│   ├── test_nodes.py
│   └── test_agent.py
└── scripts/                    # Utility scripts (indexing, scaffolding)
```

**Key Structural Rules:**
1. **Centralize State**: Define graphs' `State` in a dedicated `state.py` to prevent circular imports.
2. **Decouple Nodes**: Keep node implementations in `nodes/`. Nodes should be pure functions.
3. **Configuration**: Use `langgraph.json` at the root if deploying your graph as a service via LangSmith.

---

## Core Concepts

### 1. LangGraph Agents

LangGraph is the standard for building agents. It provides:

**Key Features:**

- **StateGraph & Functional API**: Explicit state management or code-first workflows via `@entrypoint` / `@task`.
- **Durable Execution**: Agents persist through failures and resume from checkpoints. Controllable with `durability` modes.
- **Human-in-the-Loop**: Pause and resume within any node or tool using `interrupt()`.
- **Memory**: Short-term thread memory (checkpointers) and long-term cross-session memory (`Store` API with semantic search).
- **Subgraphs**: Compose complex agents from modular sub-graphs.

**Agent Patterns:**

| Pattern | API | Use Case |
|---|---|---|
| ReAct | `create_react_agent` (LangGraph) or `create_agent` (LangChain) | Single agent with tools |
| Plan-and-Execute | Custom `StateGraph` | Separate planning and execution |
| Multi-Agent Supervisor | `StateGraph` + subgraphs | Route work across specialized agents |
| Functional Workflow | `@entrypoint` + `@task` | Code-first, linear/conditional flows |

---

### 2. State Management

LangGraph uses TypedDict for explicit state:

```python
from typing import Annotated, TypedDict
from langgraph.graph import MessagesState
from langgraph.graph.message import add_messages
from operator import add

# Minimal message-based state — use for most chat agents
class AgentState(MessagesState):
    """Extends MessagesState with custom fields."""
    retrieved_docs: list  # raw overwrite — last writer wins
    sources: Annotated[list[str], add]  # accumulate with operator.add

# Custom state for complex agents
class CustomState(TypedDict):
    messages: Annotated[list, add_messages]  # reducer: smarter message merging
    context: dict
    current_step: str
    errors: Annotated[int, lambda a, b: a + b]  # custom reducer for counting
```

**Key rule**: Fields with no reducer are **overwritten** by the last returning node. Use `Annotated[type, reducer]` whenever multiple nodes update the same field.

---

### 3. Memory Systems

Memory is separated into two tiers:

| Tier | API | Scope | Use For |
|---|---|---|---|
| Short-term | `Checkpointer` | Single `thread_id` | Conversation history, in-progress state |
| Long-term | `Store` (BaseStore) | Cross-thread / cross-user | User facts, preferences, agent instructions |

**Short-term (Checkpointers):**
- `InMemorySaver` — development/testing
- `PostgresSaver` — production (requires `langgraph-checkpoint-postgres`)
- `SqliteSaver` — production-light (requires `langgraph-checkpoint-sqlite`)

**Long-term (Stores):**
- `InMemoryStore(index={"embed": fn, "dims": N})` — development (supports vector search)
- `PostgresStore` / `RedisStore` — production
- Organise memory by namespace: `(user_id, "preferences")` or `("agent_instructions",)`
- Three conceptual types: **Semantic** (facts), **Episodic** (experiences/examples), **Procedural** (agent instructions/rules)

---

### 4. Durable Execution

Durable execution ensures agents **survive failures** by checkpointing state. It requires:
1. A **checkpointer** attached to the graph or entrypoint.
2. A **`thread_id`** in the config.
3. Wrapping side effects and non-deterministic operations in **`@task`** (Functional API) or `task()` (inside Graph API nodes).

**Durability Modes** (configurable at invoke/stream time):

```python
graph.stream({"input": "test"}, config, durability="sync")
# "exit"  — saves only on graph exit (best performance, no mid-run recovery)
# "async" — saves asynchronously during execution (good balance)
# "sync"  — saves synchronously before each step (highest durability, some overhead)
```

**Why wrap side-effects in `@task`?** When a graph resumes after a crash, nodes re-execute from their starting point. `@task` results are memoized in the checkpoint — so the API call / file write is NOT repeated.

---

### 5. Document Processing

- **Document Loaders**: Various sources (PDF, web, SQL, etc.)
- **Text Splitters**: `RecursiveCharacterTextSplitter` for general use
- **Vector Stores**: `Chroma`, `Pinecone`, `PGVector`, `Qdrant`
- **Retrievers**: `.as_retriever()`, `MultiQueryRetriever`, `EnsembleRetriever`
- **RAG Pattern**: Retrieve → Generate (see `references/architecture_patterns.md`)

---

### 6. Anthropic Claude — Key Capabilities for Agent Builders

Always leverage these when using `langchain-anthropic`:

| Feature | When to Use |
|---|---|
| **Extended Thinking** | Complex reasoning, math, multi-step planning. Supports `adaptive` mode (Opus 4.6) and `enabled` mode with `budget_tokens`. |
| **Prompt Caching** | Long system prompts, RAG contexts, or repeated tool definitions. 90% cheaper reads, 5-min TTL (1-hour available). |
| **Server Tools (Web Search)** | When the agent needs live web data. Anthropic's servers execute these automatically. |
| **MCP Connector** | Connect to remote MCP servers via HTTP without building an MCP client. |
| **Client Tools (`@tool`)** | Custom business logic executed on your infrastructure. |

**Current Claude Models (2026):**
- `claude-opus-4-6` — most powerful, adaptive thinking, 128k output tokens
- `claude-sonnet-4-6` — best balance, supports interleaved + adaptive thinking, web search
- `claude-haiku-4-5-20251001` — fastest, lowest cost

---

## Quick Start

### 1. High-Level Agent with `create_agent` (LangChain — NEW)

The newest, simplest way to build a tool-calling agent. Uses model name string, auto-detects provider.

```python
# pip install langchain "langchain[anthropic]"
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

@tool
def calculate(expression: str) -> str:
    """Evaluate a mathematical expression safely."""
    import ast, operator
    # ... safe eval logic ...
    return "42"

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[get_weather, calculate],
    system_prompt="You are a helpful assistant.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in Tokyo?"}]
})
```

---

### 2. Prebuilt ReAct Agent with `create_react_agent` (LangGraph)

For more control — exposes checkpointing, interrupt, store directly.

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

llm = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)

@tool
def search_database(query: str) -> str:
    """Search internal database for information."""
    return f"Results for: {query}"

checkpointer = MemorySaver()

agent = create_react_agent(
    llm,
    tools=[search_database],
    checkpointer=checkpointer
)

config = {"configurable": {"thread_id": "user-123"}}
result = await agent.ainvoke(
    {"messages": [("user", "Search for Python tutorials")]},
    config=config
)
```

---

### 3. Functional API Workflow (`@entrypoint` + `@task`)

Best for linear or conditional workflows where you don't want to manage explicit graph topology. Retains all LangGraph durability and HITL benefits.

```python
import uuid
from langgraph.func import entrypoint, task
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic
from typing import Any

checkpointer = InMemorySaver()
llm = ChatAnthropic(model="claude-sonnet-4-6")

@task
def do_research(topic: str) -> str:
    """Retryable, observable, memoized on resume."""
    response = llm.invoke(f"Research the topic: {topic}. Summarize in 3 bullet points.")
    return response.content

@entrypoint(checkpointer=checkpointer)
def approval_workflow(topic: str, *, previous: Any = None) -> dict:
    """Main entrypoint — supports short-term memory via `previous`."""
    research = do_research(topic).result()

    # Pause here; graph saves to checkpoint and waits
    is_approved = interrupt({"action": "review", "data": research})

    return {"research": research, "approved": is_approved}

# --- Running ---
thread_id = str(uuid.uuid4())
config = {"configurable": {"thread_id": thread_id}}

# Initial run → pauses at interrupt, emits __interrupt__ payload
for chunk in approval_workflow.stream("AI agents in 2026", config):
    print(chunk)

# Resume → provide human decision
for chunk in approval_workflow.stream(Command(resume=True), config):
    print(chunk)
```

**`entrypoint.final`** — separate the return value from what's saved to the checkpoint:

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from typing import Any

@entrypoint(checkpointer=InMemorySaver())
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # Returns `previous` to the caller, but saves `2 * number` for next run
    return entrypoint.final(value=previous, save=2 * number)
```

---

### 4. Manual StateGraph with ToolNode

Full control over graph structure — use when you need custom routing, subgraphs, or multiple agent nodes.

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return f"Results for: {query}"

tools = [search]
llm = ChatAnthropic(model="claude-sonnet-4-6").bind_tools(tools)

def agent_node(state: AgentState) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

tool_node = ToolNode(tools)

def should_continue(state: AgentState) -> str:
    last = state["messages"][-1]
    return "tools" if last.tool_calls else END

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, ["tools", END])
graph.add_edge("tools", "agent")  # Loop back

app = graph.compile()
```

---

## Advanced Topics Reference

When you need detailed implementation examples, READ these reference files FIRST:

- **`references/architecture_patterns.md`**: RAG, Multi-Agent Supervisor, Map-Reduce (`Send`), HITL with interrupts (including **in tools** and **multi-interrupt**), Memory Store with semantic search, Durable Execution `@task` inside nodes, `entrypoint.final`.
- **`references/advanced_features.md`**: Memory Management, PostgresSaver, Streaming (`v2` events), Async Execution, Anthropic **Extended Thinking**, **Server Tools** (Web Search), **Prompt Caching**, **MCP Connector** integration, Unit Testing, Performance Optimization, LangSmith observability.

---

## Anti-Patterns

### ❌ Wrapping `interrupt()` in try/except

**Why bad**: `interrupt()` raises a special `GraphInterrupt` exception internally. A broad `except Exception` will swallow it and the graph will NOT pause.

**Fix**: Leave `interrupt()` calls at the top level of your node or functional flow. Never catch generic exceptions around them.

### ❌ Performing Non-Idempotent Side-Effects Before `interrupt()`

**Why bad**: When a graph resumes after interrupt, the node **re-executes from the beginning**. A `db.create_record()` call before `interrupt()` will run twice, creating duplicate records.

**Fix**: Use upsert/idempotent operations before `interrupt()`, or place side effects *after* the interrupt call, or split into a separate node.

```python
# ✅ Good
def my_node(state):
    db.upsert_user(user_id=state["user_id"], status="pending")  # idempotent
    approved = interrupt("Approve?")
    db.create_audit_log(...)  # side effect AFTER interrupt

# ❌ Bad
def my_node(state):
    db.create_record(...)  # will duplicate on every resume!
    approved = interrupt("Approve?")
```

### ❌ Passing Non-Serializable Values to `interrupt()`

**Why bad**: Interrupt payloads are serialized to the checkpoint. Functions, class instances, etc. cannot be serialized.

**Fix**: Only pass JSON-serializable types (str, int, bool, dict, list) to `interrupt()`.

### ❌ Infinite Loop Without Exit

**Why bad**: Burns tokens, never terminates, eventually errors out.

**Fix**: Always have exit conditions:

```python
def should_continue(state):
    if state.get("iterations", 0) > 10:
        return END
    if state.get("task_complete"):
        return END
    return "agent"
```

### ❌ Giant Monolithic State

**Why bad**: Hard to reason about, unnecessary data in every node's context, serialization overhead.

**Fix**: Use input/output schemas, private state for internal data, and clear separation of concerns.

### ❌ Not Wrapping Side-Effects in `@task` for Durable Workflows

**Why bad**: If a workflow fails mid-run, the node re-executes from scratch. API calls, file writes, and DB inserts will repeat.

**Fix**: Wrap any operation with side-effects in `@task` so its result is memoized in the checkpoint.

---

## Limitations

- Visualization only available for `StateGraph` (Graph API), not Functional API
- Checkpointer required for HITL or durable execution — don't forget it in production
- State and task outputs must be JSON-serializable
- LangGraph TypeScript support is still maturing (Python is primary)

## Related Skills

Works well with: `brand-guidelines`, `autonomous-agents`, `langfuse`, `structured-output`

---

## 详细参考指南

> **遇到具体问题时，请根据下表找到对应的参考文件获取详细代码示例和完整说明。**

| 遇到的场景 | 查阅的参考文件 |
|-----------|-------------|
| 项目开始前，需要在 LangChain / LangGraph / Deep Agents 三者之间做选择 | `references/framework-selection.md` |
| 安装包、版本冲突、依赖管理（Python 或 TypeScript） | `references/langchain-dependencies.md` |
| 使用 `create_agent()`、定义 `@tool`、添加 Checkpointer | `references/langchain-fundamentals.md` |
| 需要 Human-in-the-loop 的 `HumanInTheLoopMiddleware`、结构化输出 | `references/langchain-middleware.md` |
| 构建 RAG 管道、向量存储、文档加载器、文本分割 | `references/langchain-rag.md` |
| 构建 `StateGraph`、设计节点/边、`Command`、`Send` 并行、流式输出 | `references/langgraph-fundamentals.md` |
| 实现 `interrupt()`、`Command(resume=...)`、审批工作流、验证循环 | `references/langgraph-human-in-the-loop.md` |
| Checkpointer、`thread_id`、状态历史、时间旅行、跨线程 Store 记忆 | `references/langgraph-persistence.md` |
| 使用 Deep Agents (`create_deep_agent()`)、Harness 架构、SKILL.md 格式 | `references/deep-agents-core.md` |
| Deep Agent 的记忆、文件系统后端（StateBackend / StoreBackend / CompositeBackend） | `references/deep-agents-memory.md` |
| 子 Agent 委托、TodoList 任务规划、Deep Agent 的 HITL 审批工作流 | `references/deep-agents-orchestration.md` |

同时，两个已有的参考文件：
- **`references/architecture_patterns.md`** — RAG、多 Agent 监督者、HITL 完整架构示例
- **`references/advanced_features.md`** — LangSmith、流式输出、Anthropic Extended Thinking、Prompt Caching、MCP
