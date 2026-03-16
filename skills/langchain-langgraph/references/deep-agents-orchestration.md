# deep-agents-orchestration

> 使用子 Agent、任务规划或 Deep Agents 中的人工审批时参阅此文件。涵盖 SubAgentMiddleware、用于规划的 TodoList，以及 HITL interrupts。

---

## 子 Agent（任务委托）

| 使用子 Agent 时 | 使用主 Agent 时 |
|---------------|--------------|
| 任务需要专业工具 | 通用工具已足够 |
| 希望隔离复杂工作 | 单步操作 |
| 需要为主 Agent 保持清晰上下文 | 上下文膨胀可接受 |

**工作原理：** 主 Agent 有 `task` 工具 → 创建新子 Agent → 子 Agent 自主执行 → 返回最终报告

**默认子 Agent：** "general-purpose" — 自动可用，与主 Agent 拥有相同工具/配置。

### 自定义子 Agent

**Python：**
```python
from deepagents import create_deep_agent
from langchain.tools import tool

@tool
def search_papers(query: str) -> str:
    """Search academic papers."""
    return f"Found 10 papers about {query}"

agent = create_deep_agent(
    subagents=[
        {
            "name": "researcher",
            "description": "Conduct web research and compile findings",
            "system_prompt": "Search thoroughly, return concise summary",
            "tools": [search_papers],
        }
    ]
)
# 主 Agent 委托：task(agent="researcher", instruction="Research AI trends")
```

### 带 HITL 的子 Agent

```python
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    subagents=[
        {
            "name": "code-deployer",
            "description": "Deploy code to production",
            "system_prompt": "You deploy code after tests pass.",
            "tools": [run_tests, deploy_to_prod],
            "interrupt_on": {"deploy_to_prod": True},  # 需要审批
        }
    ],
    checkpointer=MemorySaver()  # Interrupts 必须
)
```

---

## TodoList（任务规划）

| 使用 TodoList 时 | 跳过 TodoList 时 |
|---------------|---------------|
| 复杂的多步骤任务 | 简单的单操作任务 |
| 长时间运行的操作 | 快速操作（< 3 步） |

**工具签名：**
```
write_todos(todos: list[dict]) -> None
```

每个 todo 项有：
- `content`：任务描述
- `status`：`"pending"`, `"in_progress"`, `"completed"` 之一

### 示例用法

```python
from deepagents import create_deep_agent

agent = create_deep_agent()  # TodoListMiddleware 默认包含

result = agent.invoke({
    "messages": [{"role": "user", "content": "Create a REST API: design models, implement CRUD, add auth, write tests"}]
}, config={"configurable": {"thread_id": "session-1"}})

# Agent 通过 write_todos 自动规划：
# [
#   {"content": "Design data models", "status": "in_progress"},
#   {"content": "Implement CRUD endpoints", "status": "pending"},
#   {"content": "Add authentication", "status": "pending"},
#   {"content": "Write tests", "status": "pending"}
# ]
```

### 从最终状态访问 Todo

```python
result = agent.invoke({...}, config={"configurable": {"thread_id": "session-1"}})

todos = result.get("todos", [])
for todo in todos:
    print(f"[{todo['status']}] {todo['content']}")
```

---

## Human-in-the-Loop（审批工作流）

| 使用 HITL 时 | 跳过 HITL 时 |
|------------|-----------|
| 高风险操作（DB 写入、部署） | 只读操作 |
| 合规性需要人工监督 | 完全自动化的工作流 |

### 配置需要审批的工具

**Python：**
```python
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    interrupt_on={
        "write_file": True,  # 允许所有决策
        "execute_sql": {"allowed_decisions": ["approve", "reject"]},
        "read_file": False,  # 不需要中断
    },
    checkpointer=MemorySaver()  # HITL 必须
)
```

**TypeScript：**
```typescript
import { createDeepAgent } from "deepagents";
import { MemorySaver } from "@langchain/langgraph";

const agent = await createDeepAgent({
  interruptOn: {
    write_file: true,
    execute_sql: { allowedDecisions: ["approve", "reject"] },
    read_file: false,
  },
  checkpointer: new MemorySaver()  // 必须
});
```

### 完整审批工作流

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "session-1"}}

# 步骤 1：Agent 提议 write_file — 执行暂停
result = agent.invoke({
    "messages": [{"role": "user", "content": "Write config to /prod.yaml"}]
}, config=config)

# 步骤 2：检查中断状态
state = agent.get_state(config)
if state.next:
    print("有待处理的操作")

# 步骤 3：审批并恢复
result = agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config=config)
```

### 带反馈的拒绝

```python
result = agent.invoke(
    Command(resume={"decisions": [{"type": "reject", "message": "Run tests first"}]}),
    config=config,
)
```

### 执行前编辑操作

```python
result = agent.invoke(
    Command(resume={"decisions": [{
        "type": "edit",
        "edited_action": {
            "name": "execute_sql",
            "args": {"query": "DELETE FROM users WHERE last_login < '2020-01-01' LIMIT 100"},
        },
    }]}),
    config=config,
)
```

---

## 配置边界

**可以配置：**
- 子 Agent 的名称、工具、模型、系统提示
- 哪些工具需要审批
- 每个工具的允许决策类型
- TodoList 内容和结构

**不能配置：**
- 工具名称（`task`, `write_todos`）
- HITL 协议（approve/edit/reject 结构）
- 跳过 Interrupts 的 Checkpointer 要求
- 使子 Agent 有状态（它们是临时的）

---

## 常见错误与修复

### ❌ 缺少 Checkpointer
```python
# 错误
agent = create_deep_agent(interrupt_on={"write_file": True})

# 正确
agent = create_deep_agent(interrupt_on={"write_file": True}, checkpointer=MemorySaver())
```

### ❌ 子 Agent 是无状态的——缺少完整指令
```python
# 错误：子 Agent 不记得先前的调用
# task(agent='research', instruction='Find data')
# task(agent='research', instruction='What did you find?')  # 重新开始！

# 正确：前置完整指令
# task(agent='research', instruction='Find data on AI, save to /research/, return summary')
```

### ❌ Interrupts 在 invoke() 调用之间发生
```python
result = agent.invoke({...}, config=config)       # 步骤 1：触发中断
if "__interrupt__" in result:                      # 步骤 2：检查中断
    result = agent.invoke(                         # 步骤 3：恢复
        Command(resume={"decisions": [{"type": "approve"}]}),
        config=config,
    )
```

### ❌ 忘记 thread_id 用于恢复
```python
# 错误：没有 thread_id 无法恢复
agent.invoke({"messages": [...]})

# 正确
config = {"configurable": {"thread_id": "session-1"}}
agent.invoke({...}, config=config)
# 使用同一 config 的 Command 恢复
agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config=config)
```
