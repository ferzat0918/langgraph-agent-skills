# langchain-fundamentals

> 编写任何 LangChain Agent 代码时参阅此文件。涵盖 `create_agent()`、工具定义（`@tool` / `tool()`）、带 Checkpointer 的持久化、中间件模式，以及结构化输出。

## 目录
- [使用 create_agent 创建 Agent](#使用-create_agent-创建-agent)
- [定义工具](#定义工具)
- [中间件控制 Agent 行为](#中间件控制-agent-行为)
- [结构化输出](#结构化输出)
- [模型配置](#模型配置)
- [常见错误与修复](#常见错误与修复)

---

## 使用 create_agent 创建 Agent

`create_agent()` 是构建 Agent 的推荐方式。它处理 Agent 循环、工具执行和状态管理。

**所有其他替代方案都是过时的。**

| 参数 | 用途 | 示例 |
|------|------|------|
| `model` | 使用的 LLM | `"anthropic:claude-sonnet-4-5"` 或模型实例 |
| `tools` | 工具列表 | `[search, calculator]` |
| `system_prompt` / `systemPrompt` | Agent 指令 | `"You are a helpful assistant"` |
| `checkpointer` | 状态持久化 | `MemorySaver()` |
| `middleware` | 处理 hooks | `[HumanInTheLoopMiddleware(...)]` |

### 基础 Agent

**Python：**
```python
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def get_weather(location: str) -> str:
    """Get current weather for a location.

    Args:
        location: City name
    """
    return f"Weather in {location}: Sunny, 72F"

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[get_weather],
    system_prompt="You are a helpful assistant."
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in Paris?"}]
})
print(result["messages"][-1].content)
```

**TypeScript：**
```typescript
import { createAgent } from "langchain";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(
  async ({ location }) => `Weather in ${location}: Sunny, 72F`,
  {
    name: "get_weather",
    description: "Get current weather for a location.",
    schema: z.object({ location: z.string().describe("City name") }),
  }
);

const agent = createAgent({
  model: "anthropic:claude-sonnet-4-5",
  tools: [getWeather],
  systemPrompt: "You are a helpful assistant.",
});

const result = await agent.invoke({
  messages: [{ role: "user", content: "What's the weather in Paris?" }],
});
console.log(result.messages[result.messages.length - 1].content);
```

### 带持久化的 Agent

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search],
    checkpointer=checkpointer,
)

config = {"configurable": {"thread_id": "user-123"}}
agent.invoke({"messages": [{"role": "user", "content": "My name is Alice"}]}, config=config)
result = agent.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config=config)
# Agent 记得："Your name is Alice"
```

---

## 定义工具

工具是 Agent 可以调用的函数。使用 `@tool` 装饰器（Python）或 `tool()` 函数（TypeScript）。

**Python：**
```python
from langchain_core.tools import tool

@tool
def add(a: float, b: float) -> float:
    """Add two numbers.

    Args:
        a: First number
        b: Second number
    """
    return a + b
```

**TypeScript：**
```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const add = tool(
  async ({ a, b }) => a + b,
  {
    name: "add",
    description: "Add two numbers.",
    schema: z.object({
      a: z.number().describe("First number"),
      b: z.number().describe("Second number"),
    }),
  }
);
```

---

## 中间件控制 Agent 行为

中间件拦截 Agent 循环，添加人工审批、错误处理、日志记录等。

关键导入：
```python
from langchain.agents.middleware import HumanInTheLoopMiddleware, wrap_tool_call
```
```typescript
import { humanInTheLoopMiddleware, createMiddleware } from "langchain";
```

关键模式：
- **HITL**：`middleware=[HumanInTheLoopMiddleware(interrupt_on={"dangerous_tool": True})]` — 需要 `checkpointer` + `thread_id`
- **中断后恢复**：`agent.invoke(Command(resume={"decisions": [{"type": "approve"}]}), config=config)`
- **自定义中间件**：`@wrap_tool_call` 装饰器（Python）或 `createMiddleware({ wrapToolCall: ... })`（TypeScript）

> 详细的中间件示例请参见 `langchain-middleware.md`

---

## 结构化输出

使用 `response_format` 或 `with_structured_output()` 获取类型化、验证过的 Agent 响应。

**Python：**
```python
from langchain.agents import create_agent
from pydantic import BaseModel, Field

class ContactInfo(BaseModel):
    name: str
    email: str
    phone: str = Field(description="Phone number with area code")

# 选项 1：带结构化输出的 Agent
agent = create_agent(model="gpt-4.1", tools=[search], response_format=ContactInfo)
result = agent.invoke({"messages": [{"role": "user", "content": "Find contact for John"}]})
print(result["structured_response"])  # ContactInfo(name='John', ...)

# 选项 2：模型级结构化输出（无需 Agent）
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4.1")
structured_model = model.with_structured_output(ContactInfo)
response = structured_model.invoke("Extract: John, john@example.com, 555-1234")
```

**TypeScript：**
```typescript
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const ContactInfo = z.object({
  name: z.string(),
  email: z.string().email(),
  phone: z.string().describe("Phone number with area code"),
});

const model = new ChatOpenAI({ model: "gpt-4.1" });
const structuredModel = model.withStructuredOutput(ContactInfo);
const response = await structuredModel.invoke("Extract: John, john@example.com, 555-1234");
```

---

## 模型配置

`create_agent` 接受模型字符串或模型实例（用于自定义设置）：

```python
from langchain_anthropic import ChatAnthropic
agent = create_agent(model=ChatAnthropic(model="claude-sonnet-4-5", temperature=0), tools=[...])
```

---

## 常见错误与修复

### ❌ 工具描述缺失或模糊
```python
# 错误：模糊描述
@tool
def bad_tool(input: str) -> str:
    """Does stuff."""
    return "result"

# 正确：清晰、具体的描述，带 Args
@tool
def search(query: str) -> str:
    """Search the web for current information about a topic.

    Use this when you need recent data or facts.

    Args:
        query: The search query (2-10 words recommended)
    """
    return web_search(query)
```

### ❌ 没有 Checkpointer（Agent 不记得）
```python
# 错误：无持久化
agent = create_agent(model="anthropic:claude-sonnet-4-5", tools=[search])
agent.invoke({"messages": [{"role": "user", "content": "I'm Bob"}]})
agent.invoke({"messages": [{"role": "user", "content": "What's my name?"}]})
# Agent 不记得！

# 正确：添加 Checkpointer 和 thread_id
agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[search],
    checkpointer=MemorySaver(),
)
config = {"configurable": {"thread_id": "session-1"}}
agent.invoke({...}, config=config)
```

### ❌ 无限循环（缺少迭代限制）
```python
# 错误：可能无限循环
result = agent.invoke({"messages": [("user", "Do research")]})

# 正确：在 config 中设置 recursion_limit
result = agent.invoke(
    {"messages": [("user", "Do research")]},
    config={"recursion_limit": 10},
)
```

### ❌ 错误地访问结果
```python
# 错误：直接访问 result.content
result = agent.invoke({"messages": [{"role": "user", "content": "Hello"}]})
print(result.content)  # AttributeError！

# 正确：从 result dict 中访问 messages
print(result["messages"][-1].content)  # 最后一条消息内容
```
