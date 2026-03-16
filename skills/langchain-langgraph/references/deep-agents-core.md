# deep-agents-core

> 构建**任何** Deep Agents 应用时参阅此文件。涵盖 `create_deep_agent()`、Harness 架构、SKILL.md 格式和配置选项。

## 目录
- [概述](#概述)
- [何时使用 Deep Agents](#何时使用-deep-agents)
- [中间件选择](#中间件选择)
- [基础示例](#基础示例)
- [完整配置](#完整配置)
- [内置工具](#内置工具)
- [SKILL.md 格式](#skillmd-格式)
- [Skills vs Memory](#skills-vs-memory)
- [配置边界](#配置边界)
- [常见错误与修复](#常见错误与修复)

---

## 概述

Deep Agents 是建立在 LangChain/LangGraph 之上的固执己见 Agent 框架，内置中间件：

- **任务规划**：TodoListMiddleware 将复杂任务分解
- **上下文管理**：带可插拔后端的文件系统工具
- **任务委托**：SubAgent 中间件，用于生成专业 Agent
- **长期记忆**：通过 Store 实现跨线程的持久存储
- **Human-in-the-loop**：敏感操作的审批工作流
- **Skills**：按需加载专业能力

Agent Harness 自动提供这些能力——你只需配置，无需实现。

---

## 何时使用 Deep Agents

| 使用 Deep Agents 时 | 使用 LangChain 的 create_agent 时 |
|-------------------|----------------------------------|
| 多步骤任务需要规划 | 简单的单一用途任务 |
| 大型上下文需要文件管理 | 上下文适合单个提示 |
| 需要专业子 Agent | 单个 Agent 已足够 |
| 跨会话的持久记忆 | 临时的单会话工作 |

---

## 中间件选择

| 需要…… | 中间件 | 说明 |
|--------|--------|------|
| 跟踪复杂任务 | TodoListMiddleware | 默认启用 |
| 管理文件上下文 | FilesystemMiddleware | 配置 backend |
| 委托工作 | SubAgentMiddleware | 添加自定义子 Agent |
| 添加人工审批 | HumanInTheLoopMiddleware | 需要 Checkpointer |
| 加载 Skills | SkillsMiddleware | 提供 skills 目录 |
| 访问记忆 | MemoryMiddleware | 需要 Store 实例 |

---

## 基础示例

**Python：**
```python
from deepagents import create_deep_agent
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get the weather for a given city."""
    return f"It is always sunny in {city}"

agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[get_weather],
    system_prompt="You are a helpful assistant"
)

config = {"configurable": {"thread_id": "user-123"}}
result = agent.invoke({
    "messages": [{"role": "user", "content": "What's the weather in Tokyo?"}]
}, config=config)
```

**TypeScript：**
```typescript
import { createDeepAgent } from "deepagents";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(
  async ({ city }) => `It is always sunny in ${city}`,
  { name: "get_weather", description: "Get weather for a city", schema: z.object({ city: z.string() }) }
);

const agent = await createDeepAgent({
  model: "claude-sonnet-4-5-20250929",
  tools: [getWeather],
  systemPrompt: "You are a helpful assistant"
});

const config = { configurable: { thread_id: "user-123" } };
const result = await agent.invoke({
  messages: [{ role: "user", content: "What's the weather in Tokyo?" }]
}, config);
```

---

## 完整配置

包括子 Agent、Skills 和持久化的所有选项：

**Python：**
```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    name="my-assistant",
    model="claude-sonnet-4-5-20250929",
    tools=[custom_tool1, custom_tool2],
    system_prompt="Custom instructions",
    subagents=[research_agent, code_agent],
    backend=FilesystemBackend(root_dir=".", virtual_mode=True),
    interrupt_on={"write_file": True},
    skills=["./skills/"],
    checkpointer=MemorySaver(),
    store=InMemoryStore()
)
```

**TypeScript：**
```typescript
import { createDeepAgent, FilesystemBackend } from "deepagents";
import { MemorySaver, InMemoryStore } from "@langchain/langgraph";

const agent = await createDeepAgent({
  name: "my-assistant",
  model: "claude-sonnet-4-5-20250929",
  tools: [customTool1, customTool2],
  systemPrompt: "Custom instructions",
  subagents: [researchAgent, codeAgent],
  backend: new FilesystemBackend({ rootDir: ".", virtualMode: true }),
  interruptOn: { write_file: true },
  skills: ["./skills/"],
  checkpointer: new MemorySaver(),
  store: new InMemoryStore()
});
```

---

## 内置工具

每个 Deep Agent 都可以访问：

1. **规划**：`write_todos` — 跟踪多步骤任务
2. **文件系统**：`ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`
3. **委托**：`task` — 生成专业子 Agent

---

## SKILL.md 格式

Skills 使用**渐进式披露** — Agent 仅在相关时加载内容。

### 目录结构
```
skills/
└── my-skill/
    ├── SKILL.md        # 必需：主 skill 文件
    ├── examples.py     # 可选：支持文件
    └── templates/      # 可选：模板
```

### SKILL.md 格式
```markdown
---
name: my-skill
description: 清晰、具体地描述这个 skill 的功能
---

# Skill 名称

## 概述
简要解释 skill 的目的。

## 何时使用
此 skill 适用的条件。

## 说明
给 Agent 的分步指导。
```

---

## Skills vs Memory

| Skills | Memory (AGENTS.md) |
|--------|-------------------|
| 按需加载 | 启动时始终加载 |
| 特定任务的说明 | 通用偏好 |
| 大型文档 | 紧凑的上下文 |
| 目录中的 SKILL.md | 单一 AGENTS.md 文件 |

---

## 配置边界

**可以配置：**
- 模型选择和参数
- 额外的自定义工具
- 系统提示自定义
- 后端存储策略
- 哪些工具需要审批
- 带专业工具的自定义子 Agent

**不能配置：**
- 核心中间件移除（TodoList, Filesystem, SubAgent 始终存在）
- write_todos, task 或文件系统工具名称
- SKILL.md frontmatter 格式

---

## 常见错误与修复

### ❌ Interrupts 缺少 Checkpointer
```python
# 错误
agent = create_deep_agent(interrupt_on={"write_file": True})

# 正确
agent = create_deep_agent(interrupt_on={"write_file": True}, checkpointer=MemorySaver())
```

### ❌ StoreBackend 缺少 Store
```python
# 错误
agent = create_deep_agent(backend=lambda rt: StoreBackend(rt))

# 正确
agent = create_deep_agent(backend=lambda rt: StoreBackend(rt), store=InMemoryStore())
```

### ❌ 不使用 thread_id 维持对话
```python
# 错误：每次调用是独立的
agent.invoke({"messages": [{"role": "user", "content": "Hi"}]})
agent.invoke({"messages": [{"role": "user", "content": "What did I say?"}]})

# 正确
config = {"configurable": {"thread_id": "user-123"}}
agent.invoke({"messages": [...]}, config=config)
agent.invoke({"messages": [...]}, config=config)
```

### ❌ SKILL.md 缺少 Frontmatter
```markdown
<!-- 错误：缺少 frontmatter -->
# My Skill
This is my skill...

<!-- 正确：包含 YAML frontmatter -->
---
name: my-skill
description: Python testing best practices with pytest fixtures and mocking
---
# My Skill
This is my skill...
```

### ❌ Skills 缺少 Backend
```python
# 错误：没有合适 backend，skills 无法加载
agent = create_deep_agent(skills=["./skills/"])

# 正确：使用 FilesystemBackend 加载本地 skills
agent = create_deep_agent(
    backend=FilesystemBackend(root_dir=".", virtual_mode=True),
    skills=["./skills/"]
)
```

### ❌ Skill 描述过于模糊
```markdown
<!-- 错误：模糊描述 -->
---
name: helper
description: Helpful skill
---

<!-- 正确：具体描述 -->
---
name: python-testing
description: Python testing best practices with pytest fixtures, mocking, and async patterns
---
```

### ❌ 自定义子 Agent 不继承 Skills
```python
# 错误：自定义子 Agent 不继承主 Agent 的 skills
agent = create_deep_agent(
    skills=["/main-skills/"],
    subagents=[{"name": "helper", ...}]  # 无 skills
)

# 正确：显式提供 skills（通用子 Agent 会继承）
agent = create_deep_agent(
    skills=["/main-skills/"],
    subagents=[{"name": "helper", "skills": ["/helper-skills/"], ...}]
)
```
