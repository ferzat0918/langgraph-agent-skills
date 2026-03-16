# langgraph-human-in-the-loop

> 实现 Human-in-the-Loop 模式、暂停等待审批、或处理 LangGraph 中错误时参阅此文件。涵盖 `interrupt()`、`Command(resume=...)`、审批/验证工作流，以及四层错误处理策略。

## 目录
- [概述与要求](#概述与要求)
- [基础中断与恢复](#基础中断与恢复)
- [审批工作流](#审批工作流)
- [验证循环](#验证循环)
- [多个并行中断](#多个并行中断)
- [中断前的副作用必须是幂等的](#中断前的副作用必须是幂等的)
- [Subgraph 中断重新执行](#subgraph-中断重新执行)
- [常见错误与修复](#常见错误与修复)

---

## 概述与要求

LangGraph 的 Human-in-the-Loop 模式让你能够暂停图执行、向用户呈现数据，并在获得输入后恢复：

- **`interrupt(value)`** — 暂停执行，向调用者呈现一个值
- **`Command(resume=value)`** — 恢复执行，将值提供给 `interrupt()` 的返回
- **Checkpointer** — 必须：在暂停期间保存状态
- **Thread ID** — 必须：标识哪个暂停执行需要恢复

**三个必要条件：**
1. **Checkpointer** — 用 `checkpointer=InMemorySaver()`（开发）或 `PostgresSaver`（生产）编译
2. **Thread ID** — 对每次 `invoke`/`stream` 调用传入 `{"configurable": {"thread_id": "..."}}`
3. **JSON 可序列化的载荷** — 传给 `interrupt()` 的值必须是 JSON 可序列化的

---

## 基础中断与恢复

`interrupt(value)` 暂停图。值在结果的 `__interrupt__` 键下显示。`Command(resume=value)` 恢复 — resume 值成为 `interrupt()` 的返回值。

**关键点**：图恢复时，节点从**开头**重新启动 — `interrupt()` 之前的所有代码都会重新运行。

**Python：**
```python
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    approved: bool

def approval_node(state: State):
    # 暂停并请求审批
    approved = interrupt("Do you approve this action?")
    # 恢复时，Command(resume=...) 在这里返回该值
    return {"approved": approved}

checkpointer = InMemorySaver()
graph = (
    StateGraph(State)
    .add_node("approval", approval_node)
    .add_edge(START, "approval")
    .add_edge("approval", END)
    .compile(checkpointer=checkpointer)
)

config = {"configurable": {"thread_id": "thread-1"}}

# 初始运行 — 遇到 interrupt 并暂停
result = graph.invoke({"approved": False}, config)
print(result["__interrupt__"])
# [Interrupt(value='Do you approve this action?')]

# 用人工响应恢复
result = graph.invoke(Command(resume=True), config)
print(result["approved"])  # True
```

**TypeScript：**
```typescript
import { interrupt, Command, MemorySaver, StateGraph, StateSchema, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  approved: z.boolean().default(false),
});

const approvalNode = async (state: typeof State.State) => {
  const approved = interrupt("Do you approve this action?");
  return { approved };
};

const checkpointer = new MemorySaver();
const graph = new StateGraph(State)
  .addNode("approval", approvalNode)
  .addEdge(START, "approval")
  .addEdge("approval", END)
  .compile({ checkpointer });

const config = { configurable: { thread_id: "thread-1" } };

let result = await graph.invoke({ approved: false }, config);
console.log(result.__interrupt__);

result = await graph.invoke(new Command({ resume: true }), config);
console.log(result.approved);  // true
```

---

## 审批工作流

常见模式：中断显示草稿，根据人工决策路由。

```python
from langgraph.types import interrupt, Command
from langgraph.graph import StateGraph, START, END
from typing import Literal
from typing_extensions import TypedDict

class EmailAgentState(TypedDict):
    email_content: str
    draft_response: str
    classification: dict

def human_review(state: EmailAgentState) -> Command[Literal["send_reply", "__end__"]]:
    """使用 interrupt 暂停人工审阅，并根据决策路由。"""
    classification = state.get("classification", {})

    # interrupt() 必须首先执行 — 之前的任何代码都会在恢复时重新运行
    human_decision = interrupt({
        "email_id": state.get("email_content", ""),
        "draft_response": state.get("draft_response", ""),
        "urgency": classification.get("urgency"),
        "action": "Please review and approve/edit this response"
    })

    # 处理人工决策
    if human_decision.get("approved"):
        return Command(
            update={"draft_response": human_decision.get("edited_response", state.get("draft_response", ""))},
            goto="send_reply"
        )
    else:
        return Command(update={}, goto=END)
```

---

## 验证循环

在循环中使用 `interrupt()` 验证人工输入，无效时重新提示：

```python
def get_age_node(state):
    prompt = "What is your age?"

    while True:
        answer = interrupt(prompt)

        # 验证输入
        if isinstance(answer, int) and answer > 0:
            break
        else:
            # 无效输入 — 带更具体提示再次询问
            prompt = f"'{answer}' is not a valid age. Please enter a positive number."

    return {"age": answer}

# 用法：
config = {"configurable": {"thread_id": "form-1"}}
first = graph.invoke({"age": None}, config)
# __interrupt__: "What is your age?"

retry = graph.invoke(Command(resume="thirty"), config)
# __interrupt__: "'thirty' is not a valid age..."

final = graph.invoke(Command(resume=30), config)
print(final["age"])  # 30
```

---

## 多个并行中断

当并行分支各自调用 `interrupt()` 时，在单次调用中通过映射每个 interrupt ID 到其 resume 值来恢复所有：

```python
from typing import Annotated, TypedDict
import operator
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import START, END, StateGraph
from langgraph.types import Command, interrupt

class State(TypedDict):
    vals: Annotated[list[str], operator.add]

def node_a(state):
    answer = interrupt("question_a")
    return {"vals": [f"a:{answer}"]}

def node_b(state):
    answer = interrupt("question_b")
    return {"vals": [f"b:{answer}"]}

graph = (
    StateGraph(State)
    .add_node("a", node_a)
    .add_node("b", node_b)
    .add_edge(START, "a")
    .add_edge(START, "b")
    .add_edge("a", END)
    .add_edge("b", END)
    .compile(checkpointer=InMemorySaver())
)

config = {"configurable": {"thread_id": "1"}}

# 两个并行节点都遇到 interrupt() 并暂停
result = graph.invoke({"vals": []}, config)

# 一次性恢复所有待处理中断
resume_map = {
    i.id: f"answer for {i.value}"
    for i in result["__interrupt__"]
}
result = graph.invoke(Command(resume=resume_map), config)
# result["vals"] = ["a:answer for question_a", "b:answer for question_b"]
```

---

## 中断前的副作用必须是幂等的

图恢复时，节点从**开头**重新启动 — `interrupt()` 之前的**所有**代码都会重新运行。在 Subgraph 中，父节点和 Subgraph 节点都会重新执行。

**✅ 应该做：**
- 在 `interrupt()` 之前使用 **upsert**（非 insert）操作
- 使用**先检查再创建**的模式
- 尽可能将副作用放在 `interrupt()` **之后**
- 将副作用分离到它们自己的节点

**❌ 不应该做：**
- 在 `interrupt()` 之前创建新记录 — 每次恢复时都会重复
- 在 `interrupt()` 之前追加到列表 — 每次恢复时都会有重复条目

```python
# ✅ 正确：upsert 是幂等的 — 在 interrupt 之前安全
def node_a(state: State):
    db.upsert_user(user_id=state["user_id"], status="pending_approval")
    approved = interrupt("Approve this change?")
    return {"approved": approved}

# ✅ 正确：副作用在 interrupt 之后 — 只运行一次
def node_a(state: State):
    approved = interrupt("Approve this change?")
    if approved:
        db.create_audit_log(user_id=state["user_id"], action="approved")
    return {"approved": approved}

# ❌ 错误：insert 在每次恢复时都会创建重复！
def node_a(state: State):
    audit_id = db.create_audit_log({  # 恢复时再次运行！
        "user_id": state["user_id"],
        "action": "pending_approval",
    })
    approved = interrupt("Approve this change?")
    return {"approved": approved}
```

---

## Subgraph 中断重新执行

当 Subgraph 包含 `interrupt()` 时，恢复会重新执行父节点（调用 Subgraph 的）AND Subgraph 节点（调用 `interrupt()` 的）：

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- 恢复时重新执行
    subgraph_result = subgraph.invoke(some_input)
    # ...

def node_in_subgraph(state: State):
    some_other_code()  # <-- 也在恢复时重新执行
    result = interrupt("What's your name?")
    # ...
```

---

## 常见错误与修复

### ❌ 缺少 Checkpointer
```python
# 错误
graph = builder.compile()

# 正确
graph = builder.compile(checkpointer=InMemorySaver())
```

### ❌ 使用普通 dict 而非 Command 恢复
```python
# 错误
graph.invoke({"resume_data": "approve"}, config)

# 正确
graph.invoke(Command(resume="approve"), config)
```

### ❌ 恢复时使用不同的 thread_id
```python
# 错误：创建新线程而非恢复
graph.invoke(Command(resume=True), {"configurable": {"thread_id": "different-id"}})

# 正确：使用同一 config
graph.invoke(Command(resume=True), config)  # 同一个 config 对象
```

### ❌ 将 Command(update=...) 作为 invoke 输入传入
`Command(resume=...)` 是**唯一**可以作为 `invoke()`/`stream()` 输入的 Command 模式。
不要传入 `Command(update=...)` 作为输入 — 它会从最新检查点恢复，图看起来卡住了。

### ❌ 非幂等副作用在 interrupt() 之前
在 `interrupt()` 之前进行的非幂等操作（insert、append）会在每次恢复时重复。
将副作用移到 `interrupt()` **之后**，或使用 upsert/检查后再创建的模式。
