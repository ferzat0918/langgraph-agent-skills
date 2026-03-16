# langchain-middleware

> 需要 Human-in-the-loop 审批、自定义中间件或结构化输出时参阅此文件。涵盖 `HumanInTheLoopMiddleware` 人工审批危险工具调用、使用 hooks 创建自定义中间件、`Command` resume 模式，以及 Pydantic/Zod 结构化输出。

**要求：** 所有 HITL 工作流都需要 Checkpointer + thread_id 配置。

---

## Human-in-the-Loop

### 基础 HITL 设置

**Python — 在发送邮件前暂停并请求审批：**
```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import MemorySaver
from langchain.tools import tool

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email."""
    return f"Email sent to {to}"

agent = create_agent(
    model="gpt-4.1",
    tools=[send_email],
    checkpointer=MemorySaver(),  # HITL 必须
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
            }
        )
    ],
)
```

**TypeScript：**
```typescript
import { createAgent, humanInTheLoopMiddleware } from "langchain";
import { MemorySaver } from "@langchain/langgraph";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const sendEmail = tool(
  async ({ to, subject, body }) => `Email sent to ${to}`,
  {
    name: "send_email",
    description: "Send an email",
    schema: z.object({ to: z.string(), subject: z.string(), body: z.string() }),
  }
);

const agent = createAgent({
  model: "anthropic:claude-sonnet-4-5",
  tools: [sendEmail],
  checkpointer: new MemorySaver(),
  middleware: [
    humanInTheLoopMiddleware({
      interruptOn: { send_email: { allowedDecisions: ["approve", "edit", "reject"] } },
    }),
  ],
});
```

---

### 运行、检测中断、恢复

**Python：**
```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "session-1"}}

# 步骤 1：Agent 运行直到需要调用工具
result1 = agent.invoke({
    "messages": [{"role": "user", "content": "Send email to john@example.com"}]
}, config=config)

# 检测中断
if "__interrupt__" in result1:
    print(f"等待审批: {result1['__interrupt__']}")

# 步骤 2：人工审批
result2 = agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config
)
```

**TypeScript：**
```typescript
import { Command } from "@langchain/langgraph";

const config = { configurable: { thread_id: "session-1" } };

// 步骤 1
const result1 = await agent.invoke({
  messages: [{ role: "user", content: "Send email to john@example.com" }]
}, config);

// 检测中断
if (result1.__interrupt__) {
  console.log(`等待审批: ${result1.__interrupt__}`);
}

// 步骤 2：人工审批
const result2 = await agent.invoke(
  new Command({ resume: { decisions: [{ type: "approve" }] } }),
  config
);
```

---

### 编辑工具参数

```python
# 人工编辑参数 — edited_action 必须包含 name + args
result2 = agent.invoke(
    Command(resume={
        "decisions": [{
            "type": "edit",
            "edited_action": {
                "name": "send_email",
                "args": {
                    "to": "alice@company.com",  # 已修正的邮件
                    "subject": "Project Meeting - Updated",
                    "body": "...",
                },
            },
        }]
    }),
    config=config
)
```

### 带反馈的拒绝

```python
result2 = agent.invoke(
    Command(resume={
        "decisions": [{
            "type": "reject",
            "feedback": "Cannot delete customer data without manager approval",
        }]
    }),
    config=config
)
```

### 为不同工具配置不同策略

```python
agent = create_agent(
    model="gpt-4.1",
    tools=[send_email, read_email, delete_email],
    checkpointer=MemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
                "delete_email": {"allowed_decisions": ["approve", "reject"]},  # 无 edit
                "read_email": False,  # 读取无需 HITL
            }
        )
    ],
)
```

---

## 配置边界

**可以配置：**
- 哪些工具需要审批（按工具策略）
- 每个工具的允许决策（approve, edit, reject）
- 自定义中间件 hooks：`before_model`, `after_model`, `wrap_tool_call`, `before_agent`, `after_agent`
- 特定工具的中间件（仅应用于某些工具）

**不能配置：**
- 工具执行后中断（必须在之前）
- 跳过 HITL 的 Checkpointer 要求

---

## 常见错误与修复

### ❌ 缺少 Checkpointer
```python
# 错误
agent = create_agent(model="gpt-4.1", tools=[send_email], middleware=[HumanInTheLoopMiddleware({...})])

# 正确
agent = create_agent(
    model="gpt-4.1", tools=[send_email],
    checkpointer=MemorySaver(),  # 必须
    middleware=[HumanInTheLoopMiddleware({...})]
)
```

### ❌ 缺少 thread_id
```python
# 错误
agent.invoke(input)  # 无 config！

# 正确
agent.invoke(input, config={"configurable": {"thread_id": "user-123"}})
```

### ❌ 错误的 resume 语法
```python
# 错误
agent.invoke({"resume": {"decisions": [...]}})

# 正确
from langgraph.types import Command
agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config=config)
```
