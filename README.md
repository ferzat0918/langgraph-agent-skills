<div align="center">

# 🤖 LangChain Agent Builder Skills

**A production-grade skill library for building AI agents with LangChain, LangGraph & Deep Agents**

[![LangChain](https://img.shields.io/badge/LangChain-1.0_LTS-green?style=flat-square&logo=python)](https://github.com/langchain-ai/langchain)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.0-blue?style=flat-square&logo=python)](https://github.com/langchain-ai/langgraph)
[![Claude](https://img.shields.io/badge/Claude-Opus%20/%20Sonnet%204.6-orange?style=flat-square)](https://www.anthropic.com)
[![Python](https://img.shields.io/badge/Python-3.10%2B-yellow?style=flat-square&logo=python)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-purple?style=flat-square)](LICENSE)

*Stop searching documentation every time you build an agent. This is everything you need, exactly when you need it.*

<!-- keywords: langgraph agent, langchain agent builder, ai agent skills, llm agent python, langgraph tutorial, langchain skills, multi-agent system, human-in-the-loop, rag implementation, prompt engineering, mcp server, cost optimization llm, langgraph examples, ai coding assistant skills, claude anthropic agent, deep agents, create_react_agent, StateGraph -->

</div>

---

## 🧭 What Is This?

This repository is a **curated, battle-tested collection of AI coding skills** — structured knowledge files that an AI coding assistant (like Claude / Gemini) reads automatically to produce production-quality code on the first attempt.

Each skill is a **senior engineer living inside your editor** who specializes in one area. Instead of you explaining context from scratch every session, the assistant reads the relevant skill and already knows the correct APIs, proven patterns, common pitfalls, and how to combine multiple tools cleanly.

> **The result**: You go from "build me an agent" to a fully structured, memory-enabled, cost-optimized, production-ready LangGraph agent — in a single conversation.

---

## 🏗️ Core Skill: `langchain-langgraph`

> *"If you only read one skill, make it this one."*

This is the **foundational skill** of the entire repository — a comprehensive, layered reference system for the three-tier AI agent stack:

```
┌─────────────────────────────────────────┐
│              Deep Agents                │  ← Highest layer: out-of-the-box
│   (planning, memory, skills, files)     │
├─────────────────────────────────────────┤
│               LangGraph                 │  ← Orchestration layer: graphs, loops, state
│    (nodes, edges, state, persistence)   │
├─────────────────────────────────────────┤
│               LangChain                 │  ← Foundation layer: models, tools, chains
│      (models, tools, prompts, RAG)      │
└─────────────────────────────────────────┘
```

### What It Covers

| Area | Capabilities |
|------|-------------|
| **Agent Patterns** | ReAct, Plan-and-Execute, Multi-Agent Supervisor, Functional Workflow (`@entrypoint` / `@task`) |
| **State Management** | `StateGraph`, TypedDict schemas, Reducers, `MessagesState`, input/output schemas |
| **Human-in-the-Loop** | `interrupt()` / `Command(resume=...)` — node-level, tool-level, multi-step review, validation loops |
| **Memory** | Short-term (Checkpointers) + Long-term (Store API with semantic search) — 3 types: Semantic, Episodic, Procedural |
| **Durable Execution** | `@task` memoization, durability modes (`sync` / `async` / `exit`), crash recovery |
| **Parallel Processing** | Map-Reduce fan-out via `Send`, parallel `@task` execution |
| **Deep Agents** | `create_deep_agent()`, TodoListMiddleware, FilesystemMiddleware, SubAgentMiddleware, SkillsMiddleware |
| **Anthropic Claude** | Extended Thinking (adaptive + budget), Prompt Caching (90% savings), Server Tools (web search), MCP Connector |
| **Observability** | LangSmith tracing, custom callbacks, streaming (v2 events), async execution |

### Reference Architecture (13 Files, ~130KB)

The skill uses a **progressive disclosure** pattern — the main `SKILL.md` provides core concepts and quick starts, while 13 specialized reference files contain detailed code examples:

```
skills/langchain-langgraph/
├── SKILL.md (21KB)                          # Core concepts, 4 quick starts, anti-patterns
└── references/
    ├── framework-selection.md               # Decision guide: LangChain vs LangGraph vs Deep Agents
    ├── langchain-dependencies.md            # Packages, versions, templates (Python + TypeScript)
    ├── langchain-fundamentals.md            # create_agent(), @tool, Checkpointer, middleware
    ├── langchain-middleware.md              # HumanInTheLoopMiddleware, structured output
    ├── langchain-rag.md                     # Document loaders, splitters, vector stores, retrieval
    ├── langgraph-fundamentals.md            # StateGraph, nodes, edges, Command, Send, streaming
    ├── langgraph-human-in-the-loop.md       # interrupt(), approval/validation workflows, parallel HITL
    ├── langgraph-persistence.md             # Checkpointers, thread_id, time travel, Store, subgraph scoping
    ├── deep-agents-core.md                  # create_deep_agent(), Harness, SKILL.md format
    ├── deep-agents-memory.md                # StateBackend, StoreBackend, CompositeBackend
    ├── deep-agents-orchestration.md         # Sub-agents, TodoList planning, HITL approval
    ├── architecture_patterns.md (23KB)      # 13 complete patterns: RAG, Supervisor, Map-Reduce, HITL, Memory Store
    └── advanced_features.md (21KB)          # Extended Thinking, Prompt Caching, MCP, LangSmith, Testing
```

### Routing Table

When you encounter a specific problem, the skill routes you to the right reference:

| Scenario | Reference File |
|----------|---------------|
| Choosing between LangChain / LangGraph / Deep Agents | `framework-selection.md` |
| Package installation, version conflicts, dependencies | `langchain-dependencies.md` |
| Using `create_agent()`, defining `@tool`, adding Checkpointer | `langchain-fundamentals.md` |
| Human-in-the-loop middleware, structured output | `langchain-middleware.md` |
| Building RAG pipelines, vector stores, text splitting | `langchain-rag.md` |
| Building `StateGraph`, designing nodes/edges, `Command`, `Send` | `langgraph-fundamentals.md` |
| Implementing `interrupt()`, approval workflows, validation loops | `langgraph-human-in-the-loop.md` |
| Checkpointer, `thread_id`, state history, time travel, Store | `langgraph-persistence.md` |
| Using Deep Agents (`create_deep_agent()`), Harness, SKILL.md | `deep-agents-core.md` |
| Deep Agent memory, filesystem backends | `deep-agents-memory.md` |
| Sub-agent delegation, TodoList planning, Deep Agent HITL | `deep-agents-orchestration.md` |

### Representative Example

```
User: "Build me a research assistant that searches the web,
       asks for human approval before sending emails, and
       remembers user preferences across sessions."

→ Agent generates: Multi-agent graph with HITL approval,
  PostgresSaver checkpointer, InMemoryStore with semantic search,
  web_search server tool, send_email tool with interrupt(),
  all wired up and ready to run.
```

---

## 📦 Other Skills

Beyond the core engine, 8 complementary skills cover the full AI agent development stack:

| Skill | Purpose | Key Deliverables |
|-------|---------|-----------------|
| 🎯 **`prompt-engineering`** | Prompt quality & evaluation | 7-layer prompt hierarchy, Claude-specific techniques, Constitutional AI, LLM-as-Judge, A/B testing |
| 🔧 **`tool-design`** | Tool interface contracts | Consolidation principles, description engineering, error message design, MCP naming |
| 📚 **`rag-implementation`** | Knowledge retrieval | 5 RAG patterns (Hybrid, Multi-Query, HyDE, etc.), 4 vector stores, reranking, 2026 embedding models |
| 📊 **`instructor`** | Structured LLM output | Pydantic-first validation, auto-retry, streaming partial objects, multi-provider support |
| 💰 **`cost-optimization`** | Cost control | 3-tier model routing, Prompt Caching, Token Budget Guardian, context compression, Batch API |
| 🔌 **`mcp-builder`** | MCP server development | 4-phase process, tool annotations, evaluation framework, TypeScript + Python guides |
| 🚀 **`fastapi`** | API serving | Streaming endpoints (SSE, JSON Lines), dependency injection, async patterns, modern toolchain |
| 🛠️ **`skill-creator`** | Meta-tooling | Create, evaluate, and improve new skills in this format |

---

## ⚡ What You Can Build

| System | Skills Used |
|---|---|
| **Autonomous research agent** with web search, memory, and email approval | `langchain-langgraph` + `prompt-engineering` + `cost-optimization` |
| **Document Q&A chatbot** over proprietary PDFs with source citations | `langchain-langgraph` + `rag-implementation` + `prompt-engineering` |
| **Multi-agent content pipeline** (researcher → writer → reviewer) | `langchain-langgraph` + `tool-design` + `cost-optimization` |
| **Structured data extraction API** from unstructured documents | `instructor` + `fastapi` + `prompt-engineering` |
| **Agent with MCP-powered database tools** | `mcp-builder` + `langchain-langgraph` + `tool-design` |
| **Cost-aware customer support agent** with long-term user memory | All skills combined |

---

## 🏁 Getting Started

### Prerequisites

```bash
# Core (always required)
pip install langchain langchain-core langgraph langsmith

# Model provider (choose one)
pip install langchain-anthropic    # For Claude
# pip install langchain-openai     # For GPT-4o / o3

# Production persistence
pip install langgraph-checkpoint-postgres

# Optional: Deep Agents framework
pip install deepagents
```

### Environment Setup

```bash
cp .env.example .env
# Fill in your API keys:
# ANTHROPIC_API_KEY=sk-ant-...
# LANGSMITH_API_KEY=ls__...
# LANGSMITH_TRACING=true
```

### How to Use These Skills

These skills are designed to be read by AI coding assistants that support the **skills protocol** (e.g., Gemini, Claude with project files).

Point your assistant at the `skills/` directory. When you start a task, the assistant automatically finds and reads the relevant skill(s) before generating code.

> No framework? Just reference the skill files directly in your system prompt or context window.

---

## 📁 Repository Structure

```
langchain-agent-builder-skills/
│
├── README.md
├── .gitignore
│
└── skills/
    ├── langchain-langgraph/          # 🏗️ Core agent framework (21KB + 13 references)
    │   ├── SKILL.md                  #    Main skill: concepts, quick starts, anti-patterns
    │   └── references/               #    13 specialized reference files (~130KB total)
    │       ├── framework-selection.md
    │       ├── langchain-dependencies.md
    │       ├── langchain-fundamentals.md
    │       ├── langchain-middleware.md
    │       ├── langchain-rag.md
    │       ├── langgraph-fundamentals.md
    │       ├── langgraph-human-in-the-loop.md
    │       ├── langgraph-persistence.md
    │       ├── deep-agents-core.md
    │       ├── deep-agents-memory.md
    │       ├── deep-agents-orchestration.md
    │       ├── architecture_patterns.md
    │       └── advanced_features.md
    │
    ├── prompt-engineering/           # 🎯 Prompt quality & evaluation
    ├── tool-design/                  # 🔧 Tool interface contracts
    ├── rag-implementation/           # 📚 Knowledge retrieval (5 patterns)
    ├── instructor/                   # 📊 Structured LLM output
    ├── cost-optimization/            # 💰 Cost control strategies
    ├── mcp-builder/                  # 🔌 MCP server development
    ├── fastapi/                      # 🚀 API serving
    └── skill-creator/                # 🛠️ Meta-tooling
```

---

## 🔗 Related Projects & Resources

- [LangGraph](https://github.com/langchain-ai/langgraph) — The agent orchestration framework these skills are built for
- [LangChain](https://github.com/langchain-ai/langchain) — The broader ecosystem
- [Anthropic Claude](https://www.anthropic.com) — The LLM powering most examples here
- [LangSmith](https://smith.langchain.com) — Observability & evaluation for LangGraph agents
- [FastMCP](https://github.com/jlowin/fastmcp) — Python MCP server framework used in `mcp-builder`
- [Instructor](https://github.com/jxnl/instructor) — Structured outputs library used in `instructor` skill

---

## 🤝 Contributing

Found a missing pattern? Got a better example? Skills evolve.

1. Fork the repo
2. Create a branch: `git checkout -b feat/new-pattern`
3. Add your changes with clear examples and explanations
4. Open a PR with a description of what problem it solves

---

## 📄 License

MIT © [ferzat0918](https://github.com/ferzat0918)

---

<div align="center">

**If this saves you hours of documentation diving, consider giving it a ⭐**

*Built with obsessive attention to what actually works in production.*

</div>
