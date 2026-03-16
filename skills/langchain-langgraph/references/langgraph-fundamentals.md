# langgraph-fundamentals

> 编写**任何** LangGraph 代码时参阅此文件。涵盖 StateGraph、状态 Schema、节点、边、Command、Send、invoke、流式输出和错误处理。

## 目录
- [核心概念](#核心概念)
- [设计方法论](#设计方法论)
- [何时使用 LangGraph](#何时使用-langgraph)
- [状态管理](#状态管理)
- [节点](#节点)
- [边](#边)
- [Command 模式](#command-模式)
- [Send API（并行）](#send-api并行)
- [运行图：invoke 和 stream](#运行图invoke-和-stream)
- [错误处理](#错误处理)
- [常见错误与修复](#常见错误与修复)

---

## 核心概念

LangGraph 将 Agent 工作流建模为**有向图**：

- **StateGraph**：构建有状态图的主类
- **节点（Nodes）**：执行工作并更新状态的函数
- **边（Edges）**：定义执行顺序（静态或条件式）
- **START/END**：标记入口和出口的特殊节点
- **带 Reducer 的状态**：控制状态更新的合并方式

图必须先 `compile()` 才能执行。

---

## 设计方法论

构建新图时遵循以下 5 步：

1. **规划离散步骤** — 绘制工作流流程图，每个步骤对应一个节点
2. **确定每步的职责** — 分类节点：LLM 步骤、数据步骤、动作步骤或用户输入步骤
3. **设计你的状态** — 状态是所有节点的共享内存，存储原始数据
4. **构建你的节点** — 将每个步骤实现为接受状态并返回部分更新的函数
5. **连接在一起** — 用边连接节点，添加条件路由，如需要则用 Checkpointer 编译

---

## 何时使用 LangGraph

| 使用 LangGraph 时 | 使用替代方案时 |
|-----------------|-------------|
| 需要对 Agent 编排进行细粒度控制 | 快速原型 → LangChain Agent |
| 构建带分支/循环的复杂工作流 | 简单无状态工作流 → LangChain direct |
| 需要 human-in-the-loop、持久化 | 需要开箱即用功能 → Deep Agents |

---

## 状态管理

| 需求 | 解决方案 | 示例 |
|------|---------|------|
| 覆盖值 | 无 Reducer（默认） | 简单字段如计数器 |
| 追加到列表 | Reducer (operator.add / concat) | 消息历史、日志 |
| 自定义逻辑 | 自定义 Reducer 函数 | 复杂合并 |

### 状态 Schema 与 Reducer

**Python：**
```python
from typing_extensions import TypedDict, Annotated
import operator

class State(TypedDict):
    name: str  # 默认：更新时覆盖
    messages: Annotated[list, operator.add]  # 追加到列表
    total: Annotated[int, operator.add]  # 整数求和
```

**TypeScript：**
```typescript
import { StateSchema, ReducedValue, MessagesValue } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  name: z.string(),  // 默认：覆盖
  messages: MessagesValue,  // 消息内置
  items: new ReducedValue(
    z.array(z.string()).default(() => []),
    { reducer: (current, update) => current.concat(update) }
  ),
});
```

---

## 节点

节点函数接受的签名：

| 签名 | 使用时机 |
|------|---------|
| `def node(state: State)` | 只需要状态的简单节点 |
| `def node(state: State, config: RunnableConfig)` | 需要 thread_id、tags 或 configurable 值 |
| `def node(state: State, runtime: Runtime[Context])` | 需要运行时上下文、store 或 stream_writer |

```python
from langchain_core.runnables import RunnableConfig
from langgraph.runtime import Runtime

def plain_node(state: State):
    return {"results": "done"}

def node_with_config(state: State, config: RunnableConfig):
    thread_id = config["configurable"]["thread_id"]
    return {"results": f"Thread: {thread_id}"}

def node_with_runtime(state: State, runtime: Runtime[Context]):
    user_id = runtime.context.user_id
    return {"results": f"User: {user_id}"}
```

---

## 边

| 需求 | 边类型 | 使用时机 |
|------|--------|---------|
| 始终去同一节点 | `add_edge()` | 固定、确定性流 |
| 基于状态路由 | `add_conditional_edges()` | 动态分支 |
| 更新状态 AND 路由 | `Command` | 在单个节点中合并逻辑 |
| 扇出到多个节点 | `Send` | 带动态输入的并行处理 |

### 基础图（线性执行）

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    input: str
    output: str

def process_input(state: State) -> dict:
    return {"output": f"Processed: {state['input']}"}

def finalize(state: State) -> dict:
    return {"output": state["output"].upper()}

graph = (
    StateGraph(State)
    .add_node("process", process_input)
    .add_node("finalize", finalize)
    .add_edge(START, "process")
    .add_edge("process", "finalize")
    .add_edge("finalize", END)
    .compile()
)

result = graph.invoke({"input": "hello"})
print(result["output"])  # "PROCESSED: HELLO"
```

### 条件边（动态路由）

```python
from typing import Literal

class State(TypedDict):
    query: str
    route: str
    result: str

def classify(state: State) -> dict:
    if "weather" in state["query"].lower():
        return {"route": "weather"}
    return {"route": "general"}

def route_query(state: State) -> Literal["weather", "general"]:
    return state["route"]

graph = (
    StateGraph(State)
    .add_node("classify", classify)
    .add_node("weather", lambda s: {"result": "Sunny, 72F"})
    .add_node("general", lambda s: {"result": "General response"})
    .add_edge(START, "classify")
    .add_conditional_edges("classify", route_query, ["weather", "general"])
    .add_edge("weather", END)
    .add_edge("general", END)
    .compile()
)
```

---

## Command 模式

Command 在单个返回值中结合状态更新和路由：
- **`update`**：要应用的状态更新（类似于从节点返回 dict）
- **`goto`**：下一步要导航到的节点名称
- **`resume`**：`interrupt()` 后恢复的值 — 见 `langgraph-human-in-the-loop.md`

```python
from langgraph.types import Command
from typing import Literal

class State(TypedDict):
    count: int
    result: str

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    """在一次返回中更新状态 AND 决定下一个节点。"""
    new_count = state["count"] + 1
    if new_count > 5:
        return Command(update={"count": new_count}, goto="node_c")
    return Command(update={"count": new_count}, goto="node_b")
```

> ⚠️ **警告**：`Command` 只添加**动态**边 — 用 `add_edge` 定义的静态边仍然会执行。如果 `node_a` 返回 `Command(goto="node_c")` 且你也有 `graph.add_edge("node_a", "node_b")`，则 `node_b` 和 `node_c` **都会运行**。

---

## Send API（并行）

扇出到并行 Worker：从条件边返回 `[Send("worker", {...})]`。结果字段需要 Reducer。

```python
from langgraph.types import Send
from typing import Annotated
import operator

class OrchestratorState(TypedDict):
    tasks: list[str]
    results: Annotated[list, operator.add]  # Reducer 必须
    summary: str

def orchestrator(state: OrchestratorState):
    """扇出任务到 Worker。"""
    return [Send("worker", {"task": task}) for task in state["tasks"]]

def worker(state: dict) -> dict:
    return {"results": [f"Completed: {state['task']}"]}

def synthesize(state: OrchestratorState) -> dict:
    return {"summary": f"Processed {len(state['results'])} tasks"}

graph = (
    StateGraph(OrchestratorState)
    .add_node("worker", worker)
    .add_node("synthesize", synthesize)
    .add_conditional_edges(START, orchestrator, ["worker"])
    .add_edge("worker", "synthesize")
    .add_edge("synthesize", END)
    .compile()
)

result = graph.invoke({"tasks": ["Task A", "Task B", "Task C"]})
```

---

## 运行图：invoke 和 stream

### invoke 基础

```python
result = graph.invoke({"input": "hello"})
# 带 config（用于持久化、tags 等）
result = graph.invoke({"input": "hello"}, {"configurable": {"thread_id": "1"}})
```

### stream 模式选择

| 模式 | 流式内容 | 使用场景 |
|------|---------|---------|
| `values` | 每步后的完整状态 | 监控完整状态 |
| `updates` | 状态增量 | 跟踪增量更新 |
| `messages` | LLM tokens + 元数据 | 聊天 UI |
| `custom` | 用户定义的数据 | 进度指示器 |

### 流式 LLM Tokens

```python
for chunk in graph.stream(
    {"messages": [HumanMessage("Hello")]},
    stream_mode="messages"
):
    token, metadata = chunk
    if hasattr(token, "content"):
        print(token.content, end="", flush=True)
```

### 自定义流式数据

```python
from langgraph.config import get_stream_writer

def my_node(state):
    writer = get_stream_writer()
    writer("Processing step 1...")
    # 做工作
    writer("Complete!")
    return {"result": "done"}

for chunk in graph.stream({"data": "test"}, stream_mode="custom"):
    print(chunk)
```

---

## 错误处理

| 错误类型 | 谁来修复 | 策略 | 示例 |
|---------|---------|------|------|
| 瞬时性（网络、限速） | 系统 | `RetryPolicy(max_attempts=3)` | `add_node(..., retry_policy=...)` |
| LLM 可恢复（工具失败） | LLM | `ToolNode(tools, handle_tool_errors=True)` | 错误作为 ToolMessage 返回 |
| 用户可修复（缺少信息） | 人工 | `interrupt({...})` | 收集缺失数据（见 HITL 参考） |
| 意外错误 | 开发者 | 让它冒泡 | `raise` |

```python
from langgraph.types import RetryPolicy

# 重试策略
workflow.add_node(
    "search_documentation",
    search_documentation,
    retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0)
)

# 工具节点错误处理
from langgraph.prebuilt import ToolNode
tool_node = ToolNode(tools, handle_tool_errors=True)
workflow.add_node("tools", tool_node)
```

---

## 常见错误与修复

### ❌ 忘记调用 compile()
```python
# 错误
builder.invoke({"input": "test"})  # AttributeError！

# 正确
graph = builder.compile()
graph.invoke({"input": "test"})
```

### ❌ 列表字段没有 Reducer
```python
# 错误：列表将被覆盖
class State(TypedDict):
    messages: list  # 没有 Reducer！
# 节点 1 返回：{"messages": ["A"]}
# 节点 2 返回：{"messages": ["B"]}
# 最终：{"messages": ["B"]}  # "A" 已丢失！

# 正确：使用带 operator.add 的 Annotated
class State(TypedDict):
    messages: Annotated[list, operator.add]
# 最终：{"messages": ["A", "B"]}
```

### ❌ 节点返回整个状态而非部分更新
```python
# 错误：返回整个状态对象
def my_node(state: State) -> State:
    state["field"] = "updated"
    return state  # 不要这样！

# 正确：只返回包含更新的 dict
def my_node(state: State) -> dict:
    return {"field": "updated"}
```

### ❌ 无限循环（缺少出口条件）
```python
# 错误：永远循环
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", "node_a")

# 正确
def should_continue(state):
    return END if state["count"] > 10 else "node_b"
builder.add_conditional_edges("node_a", should_continue)
```

### ❌ 其他常见错误
```python
# 路由器必须返回图中存在的节点名称
builder.add_node("my_node", func)  # 在边中引用之前先添加节点
builder.add_conditional_edges("node_a", router, ["my_node"])

# Command 返回类型需要 Literal 声明路由目标（Python）
def node_a(state) -> Command[Literal["node_b", "node_c"]]:
    return Command(goto="node_b")

# START 只是入口 - 不能路由回去
builder.add_edge("node_a", START)  # 错误！
builder.add_edge("node_a", "entry")  # 使用命名入口节点代替

# Send 并行结果需要 Reducer
class State(TypedDict):
    results: Annotated[list, operator.add]  # 列表 Reducer，不是字符串
```
