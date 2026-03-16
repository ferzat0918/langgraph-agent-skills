# framework-selection

> 在任何 LangChain/LangGraph/Deep Agents 项目**开始前**调用此参考。用于决策使用哪个框架层：LangChain、LangGraph、Deep Agents 或组合。**必须在使用其他技能前先查阅本文件。**

LangChain、LangGraph 和 Deep Agents 是**分层**关系，而非竞争关系。每一层都建立在下一层之上：

```
┌─────────────────────────────────────────┐
│              Deep Agents                │  ← 最高层：开箱即用
│   (planning, memory, skills, files)     │
├─────────────────────────────────────────┤
│               LangGraph                 │  ← 编排层：图、循环、状态
│    (nodes, edges, state, persistence)   │
├─────────────────────────────────────────┤
│               LangChain                 │  ← 基础层：模型、工具、链
│      (models, tools, prompts, RAG)      │
└─────────────────────────────────────────┘
```

选择更高层不会切断对底层的访问——你可以在 Deep Agents 中使用 LangGraph 图，也可以在两者中使用 LangChain 基础组件。

---

## 决策指南

按顺序回答以下问题：

| 问题 | 是 → | 否 → |
|------|------|------|
| 任务是否需要分解子任务、跨长会话管理文件、持久记忆或按需加载技能？ | **Deep Agents** | ↓ |
| 任务是否需要复杂的控制流——循环、动态分支、并行工作者、human-in-the-loop 或自定义状态？ | **LangGraph** | ↓ |
| 这是单一用途的 Agent，接受输入、执行工具并返回结果？ | **LangChain** (`create_agent`) | ↓ |
| 这是纯粹的模型调用、链或带有 Agent 循环的检索管道？ | **LangChain** (LCEL / chain) | — |

---

## 框架概述

### LangChain — 任务聚焦且自包含时使用

**最适合：**
- 使用固定工具集的单一用途 Agent
- RAG 管道和文档问答
- 模型调用、提示模板、输出解析
- 快速原型，Agent 逻辑简单

**不适合：**
- Agent 需要跨多步骤规划
- 状态需要跨多个会话持久化
- 控制流是条件性或迭代性的

**接下来调用的参考：** `langchain-fundamentals`, `langchain-rag`, `langchain-middleware`

---

### LangGraph — 需要控制流所有权时使用

**最适合：**
- 带有分支逻辑或循环的 Agent（例如重试直到正确、反思）
- 多步骤工作流，不同路径取决于中间结果
- 特定步骤的 Human-in-the-loop 审批
- 并行扇出 / 扇入（Map-Reduce 模式）
- 会话内跨调用的持久状态

**不适合：**
- 希望规划、文件管理和子 Agent 委托自动处理（使用 Deep Agents）
- 工作流本身已足够简单

**接下来调用的参考：** `langgraph-fundamentals`, `langgraph-human-in-the-loop`, `langgraph-persistence`

---

### Deep Agents — 任务开放且多维时使用

**最适合：**
- 需要将工作分解为 Todo 列表的长时间运行任务
- 需要读、写和管理跨会话文件的 Agent
- 将子任务委托给专门的子 Agent
- 按需加载领域特定技能
- 跨多个会话存活的持久记忆

**内置中间件：**

| 中间件 | 提供的能力 | 是否常开？ |
|--------|-----------|-----------|
| `TodoListMiddleware` | `write_todos` 工具 | ✓ |
| `FilesystemMiddleware` | `ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep` | ✓ |
| `SubAgentMiddleware` | `task` 工具 | ✓ |
| `SkillsMiddleware` | 从 skills 目录按需加载 SKILL.md | 可选 |
| `MemoryMiddleware` | 通过 Store 实现跨会话长期记忆 | 可选 |
| `HumanInTheLoopMiddleware` | 在敏感工具调用前暂停并请求审批 | 可选 |

**接下来调用的参考：** `deep-agents-core`, `deep-agents-memory`, `deep-agents-orchestration`

---

## 混合使用

因为框架是分层的，可以在同一项目中组合使用。最常见的模式是以 Deep Agents 作为顶层编排器，针对专门子 Agent 使用 LangGraph。

| 场景 | 推荐模式 |
|------|---------|
| 主 Agent 需要规划和记忆，但某个子任务需要精确图控制 | Deep Agents 编排器 → LangGraph 子 Agent |
| 专门管道（如 RAG、反思循环）由更广泛的 Agent 调用 | LangGraph 图包装为工具或子 Agent |
| 高级协调 + 特定领域的底层图 | Deep Agents + LangGraph 编译图作为子 Agent |

---

## 快速参考对比

| | LangChain | LangGraph | Deep Agents |
|---|-----------|-----------|-------------|
| **控制流** | 固定（工具循环） | 自定义（图） | 托管（中间件） |
| **规划** | ✗ | 手动 | ✓ TodoListMiddleware |
| **文件管理** | ✗ | 手动 | ✓ FilesystemMiddleware |
| **持久记忆** | ✗ | 需要 Checkpointer | ✓ MemoryMiddleware |
| **子 Agent 委托** | ✗ | 手动 | ✓ SubAgentMiddleware |
| **Human-in-the-loop** | ✗ | 手动 interrupt | ✓ HumanInTheLoopMiddleware |
| **自定义图边** | ✗ | ✓ 完全控制 | 有限 |
| **设置复杂度** | 低 | 中 | 低 |
| **灵活性** | 中 | 高 | 中 |
