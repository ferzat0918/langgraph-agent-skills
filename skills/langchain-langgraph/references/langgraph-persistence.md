# langgraph-persistence

> LangGraph 需要状态持久化、记住对话、浏览历史记录，或配置 Subgraph Checkpointer 作用域时参阅此文件。涵盖 Checkpointers、thread_id、时间旅行、Store 和 Subgraph 持久化模式。

## 目录
- [概述](#概述)
- [Checkpointer 选择](#checkpointer-选择)
- [Checkpointer 设置](#checkpointer-设置)
- [线程管理](#线程管理)
- [状态历史与时间旅行](#状态历史与时间旅行)
- [Subgraph Checkpointer 作用域](#subgraph-checkpointer-作用域)
- [长期记忆（Store）](#长期记忆store)
- [常见错误与修复](#常见错误与修复)

---

## 概述

LangGraph 的持久化层通过在每个超级步骤检查点图状态来实现持久执行：

- **Checkpointer**：在每个超级步骤保存/加载图状态
- **Thread ID**：标识独立的检查点序列（对话）
- **Store**：跨线程记忆，用于用户偏好、事实

**两种记忆类型：**
- **短期**（checkpointer）：线程范围的对话历史
- **长期**（store）：跨线程的用户偏好、事实

---

## Checkpointer 选择

| Checkpointer | 使用场景 | 生产可用 |
|-------------|---------|---------|
| `InMemorySaver` | 测试、开发 | 否 |
| `SqliteSaver` | 本地开发 | 部分 |
| `PostgresSaver` | 生产 | 是 |

---

## Checkpointer 设置

### 基础持久化（开发）

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict, Annotated
import operator

class State(TypedDict):
    messages: Annotated[list, operator.add]

def add_message(state: State) -> dict:
    return {"messages": ["Bot response"]}

checkpointer = InMemorySaver()

graph = (
    StateGraph(State)
    .add_node("respond", add_message)
    .add_edge(START, "respond")
    .add_edge("respond", END)
    .compile(checkpointer=checkpointer)  # 在编译时传入
)

# 始终提供 thread_id
config = {"configurable": {"thread_id": "conversation-1"}}

result1 = graph.invoke({"messages": ["Hello"]}, config)
print(len(result1["messages"]))  # 2

result2 = graph.invoke({"messages": ["How are you?"]}, config)
print(len(result2["messages"]))  # 4（之前的 + 新的）
```

### 生产 PostgreSQL

```python
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
) as checkpointer:
    checkpointer.setup()  # 仅第一次使用时创建表
    graph = builder.compile(checkpointer=checkpointer)
```

**TypeScript：**
```typescript
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const checkpointer = PostgresSaver.fromConnString(
  "postgresql://user:pass@localhost/db"
);
await checkpointer.setup(); // 仅第一次使用时创建表

const graph = builder.compile({ checkpointer });
```

---

## 线程管理

不同的 `thread_id` 维护独立的状态：

```python
alice_config = {"configurable": {"thread_id": "user-alice"}}
bob_config = {"configurable": {"thread_id": "user-bob"}}

graph.invoke({"messages": ["Hi from Alice"]}, alice_config)
graph.invoke({"messages": ["Hi from Bob"]}, bob_config)

# Alice 的状态与 Bob 的隔离
```

---

## 状态历史与时间旅行

```python
config = {"configurable": {"thread_id": "session-1"}}

result = graph.invoke({"messages": ["start"]}, config)

# 浏览检查点历史
states = list(graph.get_state_history(config))

# 从过去的检查点重放
past = states[-2]
result = graph.invoke(None, past.config)  # None = 从检查点恢复

# 或分叉：在过去的检查点更新状态，然后恢复
fork_config = graph.update_state(past.config, {"messages": ["edited"]})
result = graph.invoke(None, fork_config)
```

### 手动更新状态

```python
config = {"configurable": {"thread_id": "session-1"}}

# 恢复前修改状态
graph.update_state(config, {"data": "manually_updated"})

# 用更新后的状态恢复
result = graph.invoke(None, config)
```

---

## Subgraph Checkpointer 作用域

编译 Subgraph 时，`checkpointer` 参数控制持久化行为。

| 功能 | `checkpointer=False` | `None`（默认） | `True` |
|-----|---------------------|--------------|--------|
| Interrupts (HITL) | 否 | 是 | 是 |
| 多轮记忆 | 否 | 否 | 是 |
| 多次调用（不同 Subgraph） | 是 | 是 | 注意（可能命名空间冲突） |
| 多次调用（同一 Subgraph） | 是 | 是 | 否 |
| 状态检查 | 否 | 注意（仅当前调用） | 是 |

**何时使用各模式：**

- **`checkpointer=False`** — Subgraph 不需要 interrupt 或持久化。最简单，无检查点开销。
- **`None`（默认/省略 `checkpointer`）** — Subgraph 需要 `interrupt()` 但不需要多轮记忆。每次调用重新开始，但可以暂停/恢复。
- **`checkpointer=True`** — Subgraph 需要跨调用记住状态（多轮对话）。

```python
# 不需要 interrupt — 退出检查点
subgraph = subgraph_builder.compile(checkpointer=False)

# 需要 interrupt 但不需要跨调用持久化（默认）
subgraph = subgraph_builder.compile()

# 需要跨调用持久化（有状态）
subgraph = subgraph_builder.compile(checkpointer=True)
```

> ⚠️ **警告**：有状态 Subgraph（`checkpointer=True`）不支持在单个节点中多次调用同一 Subgraph 实例 — 这些调用写入同一检查点命名空间，会产生冲突。

### 并行 Subgraph 命名空间隔离

当多个**不同**有状态 Subgraph 并行运行时，将每个包装在带唯一节点名的 `StateGraph` 中以稳定命名空间隔离：

```python
from langgraph.graph import MessagesState, StateGraph

def create_sub_agent(model, *, name, **kwargs):
    """用唯一节点名包装 Agent 以进行命名空间隔离。"""
    agent = create_agent(model=model, name=name, **kwargs)
    return (
        StateGraph(MessagesState)
        .add_node(name, agent)  # 唯一名称 -> 稳定命名空间
        .add_edge("__start__", name)
        .compile()
    )

fruit_agent = create_sub_agent("gpt-4.1-mini", name="fruit_agent", tools=[fruit_info], checkpointer=True)
veggie_agent = create_sub_agent("gpt-4.1-mini", name="veggie_agent", tools=[veggie_info], checkpointer=True)
```

---

## 长期记忆（Store）

**短期 vs 长期记忆：**
- Checkpointer（短期）：线程范围，对话历史
- Store（长期）：**跨线程**，用户偏好、事实

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# 保存用户偏好（在所有线程中可用）
store.put(("alice", "preferences"), "language", {"preference": "short responses"})

# 在节点中使用 Store — 通过 runtime 访问
from langgraph.runtime import Runtime

def respond(state, runtime: Runtime):
    prefs = runtime.store.get((state["user_id"], "preferences"), "language")
    return {"response": f"Using preference: {prefs.value}"}

# 同时使用 checkpointer 和 store 编译
graph = builder.compile(checkpointer=checkpointer, store=store)

# 两个线程访问同一长期记忆
graph.invoke({"user_id": "alice"}, {"configurable": {"thread_id": "thread-1"}})
graph.invoke({"user_id": "alice"}, {"configurable": {"thread_id": "thread-2"}})  # 同样的偏好！
```

### Store 基础操作

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

store.put(("user-123", "facts"), "location", {"city": "San Francisco"})  # Put
item = store.get(("user-123", "facts"), "location")  # Get
results = store.search(("user-123", "facts"), filter={"city": "San Francisco"})  # Search
store.delete(("user-123", "facts"), "location")  # Delete
```

---

## 常见错误与修复

### ❌ 缺少 thread_id
```python
# 错误：状态不会持久化！
graph.invoke({"messages": ["Hello"]})
graph.invoke({"messages": ["What did I say?"]})  # 不记得！

# 正确：始终提供 thread_id
config = {"configurable": {"thread_id": "session-1"}}
graph.invoke({"messages": ["Hello"]}, config)
graph.invoke({"messages": ["What did I say?"]}, config)  # 记得！
```

### ❌ 生产使用 InMemorySaver
```python
# 错误：进程重启后数据丢失
checkpointer = InMemorySaver()

# 正确：生产使用持久化存储
from langgraph.checkpoint.postgres import PostgresSaver
with PostgresSaver.from_conn_string("postgresql://...") as checkpointer:
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)
```

### ❌ update_state 不绕过 Reducer
```python
from langgraph.types import Overwrite

# State 有 Reducer：items: Annotated[list, operator.add]
# 当前状态：{"items": ["A", "B"]}

# update_state 通过 Reducer 处理
graph.update_state(config, {"items": ["C"]})  # 结果：["A", "B", "C"] — 追加了！

# 要替换而非追加，使用 Overwrite
graph.update_state(config, {"items": Overwrite(["C"])})  # 结果：["C"] — 替换了
```

### ❌ 在节点中直接访问 Store
```python
# 错误：Store 在节点中不可用
def my_node(state):
    store.put(...)  # NameError！store 未定义

# 正确：通过 Runtime 对象访问 Store
from langgraph.runtime import Runtime

def my_node(state, runtime: Runtime):
    runtime.store.put(...)  # 正确的 Store 实例
```

### ❌ 在单节点中并行运行同一有状态 Subgraph
有状态 Subgraph（`checkpointer=True`）不能在单个节点中被调用多次 — 会产生命名空间冲突。
为每个并行实例使用不同的 Subgraph 对象，或使用命名空间包装器。
