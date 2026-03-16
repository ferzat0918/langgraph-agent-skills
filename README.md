<div align="center">

# 🤖 LangGraph Agent Builder Skills

**A production-grade skill library for building AI agents with LangGraph, LangChain & Claude**

[![LangGraph](https://img.shields.io/badge/LangGraph-0.3.x-blue?style=flat-square&logo=python)](https://github.com/langchain-ai/langgraph)
[![LangChain](https://img.shields.io/badge/LangChain-0.3.x-green?style=flat-square&logo=python)](https://github.com/langchain-ai/langchain)
[![Claude](https://img.shields.io/badge/Claude-Sonnet%204.6-orange?style=flat-square)](https://www.anthropic.com)
[![Python](https://img.shields.io/badge/Python-3.9%2B-yellow?style=flat-square&logo=python)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-purple?style=flat-square)](LICENSE)
[![Stars](https://img.shields.io/github/stars/ferzat0918/langchain-agent-builer-skills?style=flat-square)](https://github.com/ferzat0918/langchain-agent-builer-skills)

*Stop searching documentation every time you build. This is everything you need, exactly when you need it.*

<!-- keywords: langgraph agent, langchain agent builder, ai agent skills, llm agent python, langgraph tutorial, langchain skills, multi-agent system, human-in-the-loop, rag implementation, prompt engineering, mcp server, cost optimization llm, langgraph examples, ai coding assistant skills, claude anthropic agent -->

</div>

---

## 🧭 What Is This?

This repository is a **curated, battle-tested collection of AI coding skills** — structured knowledge files that an AI coding assistant (like Claude / Antigravity) reads automatically to give you production-quality code the first time.

Think of each skill as a **senior engineer living inside your editor** who specializes in one area. Instead of you explaining context from scratch every session, the assistant reads the relevant skill and already knows:

- Which APIs to use (and which to avoid)
- The exact patterns that work in production
- The common pitfalls and how to prevent them
- How to combine multiple tools cleanly

> **The result**: You go from "write me an agent" to a fully structured, memory-enabled, cost-optimized, production-ready LangGraph agent — in a single conversation.

---

## 🗺️ The Big Picture

These 9 skills are designed to work together as a **complete AI agent development stack**:

```
┌─────────────────────────────────────────────────────────────────┐
│                      YOUR AI AGENT SYSTEM                       │
├──────────────┬──────────────┬───────────────┬───────────────────┤
│  🧠 Agent    │  📚 Knowledge │  🔧 Tools     │  🚀 Serving       │
│   Logic      │    Layer      │   & Output    │   & Ops           │
├──────────────┼──────────────┼───────────────┼───────────────────┤
│ langchain-   │ rag-          │ tool-design   │ fastapi           │
│ langgraph    │ implementation│               │                   │
│              │               │ instructor    │ cost-optimization │
│ prompt-      │ mcp-builder   │               │                   │
│ engineering  │               │               │                   │
└──────────────┴──────────────┴───────────────┴───────────────────┘
                        ↑ all powered by ↑
              skill-creator  (build your own skills)
```

---

## 📦 Skills Reference

### 🏗️ `langchain-langgraph` — The Core Engine
> *"If you only read one skill, make it this one."*

The foundational skill for everything. Covers the complete LangGraph API surface with production-grade patterns.

**What it gives you:**
- `StateGraph` + `Functional API (@entrypoint / @task)` with working examples
- 4 agent patterns: **ReAct**, **Plan-and-Execute**, **Multi-Agent Supervisor**, **Functional Workflow**
- **Human-in-the-Loop** with `interrupt()` in 3 modes: node-level, tool-level, and multi-step review
- **Dual memory system**: Short-term (Checkpointers) + Long-term (Memory Store with semantic search)
- **Durable execution**: agents that survive crashes and resume mid-run
- `Map-Reduce` parallel fan-out using `Send`
- Full **Anthropic integration**: Extended Thinking, Prompt Caching, Server Tools, MCP Connector
- LangSmith observability, streaming (v2 events), async execution
- 13 architecture patterns in `references/` with complete, copy-paste-ready code
- 5 documented anti-patterns with before/after fixes

**Representative capability:**
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

### 🎯 `prompt-engineering` — The Quality Multiplier
> *"The #1 lever for agent quality — better than a bigger model."*

Most agents fail not because of code bugs, but because of bad prompts. This skill teaches you to engineer prompts like a professional.

**What it gives you:**
- **Agent Prompt Hierarchy**: A 7-layer template (Role → Task → Context → Tools → Examples → Format → Constitutional Rules)
- **Claude-specific techniques**: XML tags, `<thinking>` blocks, prefilling, signed reasoning
- **Tool Description Engineering**: The 4-question rule (What / When / Accepts / Returns)
- **Chain-of-Thought & ReAct** patterns with templates
- **Few-shot example design**: How to write examples that actually improve behavior (with negative examples)
- **Anti-hallucination system**: Epistemic hedging + grounded RAG response templates
- **Constitutional AI self-correction loop**: Generate → Critique → Revise
- **LLM-as-a-Judge evaluation**: Direct scoring + pairwise comparison with bias mitigation
- **A/B testing framework** with Cohen's d effect size for statistical rigor
- **Debugging framework**: Symptom → Root Cause → Fix for 6 common agent failures

**Representative capability:**
```
User: "The agent keeps using the wrong tool and hallucinating."

→ Skill diagnoses: Tool descriptions lack explicit triggers.
  Generates: Rewritten descriptions, a self-verification
  checkpoint, and a grounded response template. Sets up
  LLM-as-Judge to measure improvement before/after.
```

---

### 🔧 `tool-design` — The Contract Layer
> *"If a human engineer can't decide which tool to use, neither can the agent."*

Tools are the interface between your deterministic systems and a non-deterministic LLM. This skill teaches you to design that interface correctly.

**What it gives you:**
- **The Consolidation Principle**: When to merge tools, when to keep them separate
- **Architectural Reduction**: How stripping tools down to primitives often *outperforms* complex scaffolding
- **Tool description engineering**: 4-question framework with good/bad examples
- **Response format optimization**: Concise vs. detailed mode, token-efficient returns
- **Error message design**: Messages that enable agent self-recovery
- **MCP tool naming conventions**: Fully-qualified `ServerName:tool_name` format
- **The tool-testing loop**: Use an agent to diagnose its own tool failures and rewrite descriptions (40% task completion improvement in production tests)

---

### 📚 `rag-implementation` — The Knowledge Layer
> *"Give your agent a brain that knows things you never put in its training."*

Retrieval-Augmented Generation turns your documents, databases, and knowledge bases into something the agent can actually reason over.

**What it gives you:**
- **5 advanced RAG patterns**:
  - `Hybrid Search` (BM25 + semantic, Reciprocal Rank Fusion)
  - `Multi-Query` (generate query variants for better recall)
  - `Contextual Compression` (extract only the relevant passage)
  - `Parent Document Retriever` (precise child chunks, full parent context)
  - `HyDE` (Hypothetical Document Embeddings for dense-to-dense search)
- **4 vector stores** with full configuration: Pinecone, Weaviate, Chroma, pgvector
- **4 chunking strategies**: Recursive, Token-based, Semantic, Markdown-header-aware
- **Reranking**: Cross-encoder (ms-marco) + Cohere Rerank + MMR diversity
- **2026 embedding models**: `voyage-3-large` (Anthropic-recommended), `text-embedding-3-large`, multilingual
- **Evaluation metrics**: retrieval precision/recall, faithfulness, context relevance
- All examples **natively integrated with LangGraph StateGraph**

---

### 📊 `instructor` — Structured Output Enforcer
> *"Stop parsing LLM responses manually. Get typed Python objects every time."*

When your agent needs to return structured data — classifications, extracted entities, formatted reports — `instructor` wraps the LLM call in Pydantic validation with automatic retry.

**What it gives you:**
- **Pydantic-first**: Define your output schema, get a validated Python object back — guaranteed
- **Auto-retry**: If the LLM returns invalid data, instructor sends the validation error back and tries again (up to N times)
- **Streaming**: `create_partial()` for real-time partial object updates, `create_iterable()` for streaming lists
- **Multi-provider**: Anthropic, OpenAI, Ollama — same API
- **5 production patterns**: Data extraction, Classification (Enum), Multi-entity, Structured analysis, Batch processing
- Union types, nested models, dynamic model creation, custom validators
- Comparison table vs. LangChain structured output, DSPy, manual JSON

---

### 💰 `cost-optimization` — The Savings Engine
> *"Default to Opus for everything and watch your bill hit $10,000/month. Or read this."*

Production agents can be 10-20x cheaper with the right strategies. This skill gives you a systematic playbook.

**What it gives you:**
- **3-tier model routing**: Classify complexity with Haiku ($0.80/M), route to Sonnet or Opus only when needed. Typical result: 70% of tasks served by Haiku.
- **Prompt Caching**: 90% savings on repeated large contexts. 3-level strategy: system prompt → RAG documents → dynamic conversation
- **Token Budget Guardian**: A LangGraph node that monitors spend in real-time, downgrades model tier at 50%/80% thresholds, hard-stops at 100%
- **Context Compression**: Rolling summary (60-80% reduction) + LLMLingua for RAG context (70% reduction)
- **Batch API**: 50% discount for offline/async workloads
- **Output control**: `max_tokens` limits + structured output (3x fewer output tokens vs. prose)
- **Cost reduction playbook**: 3-phase checklist (Quick Wins → Medium Effort → Advanced), typical results: 3-5x → 5-10x → 10-20x savings
- **7 anti-patterns** with quantified cost impact (e.g., "No system prompt caching = 6x overspend")

---

### 🔌 `mcp-builder` — The Integration Bridge
> *"Connect your agent to any external service without writing a custom client."*

Model Context Protocol (MCP) lets Claude call tools on remote servers without you implementing the client side. This skill teaches you to build those servers.

**What it gives you:**
- **4-phase development process**: Research → Implement → Review/Test → Evaluate
- TypeScript (recommended) and Python (FastMCP) implementation guides
- Tool annotation system: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`
- **Evaluation framework**: Generate 10 complex, verifiable QA pairs to test your server's real-world effectiveness
- MCP Inspector integration for local testing
- Best practices for: tool naming, pagination, error messages, response format (JSON vs Markdown)
- How to connect to your MCP server from a LangGraph agent via the Anthropic MCP Connector

---

### 🚀 `fastapi` — The Deployment Layer
> *"Your agent is useless if nobody can call it."*

Once your agent is built, you need to expose it as a reliable API. This skill keeps your FastAPI code clean, modern, and performant.

**What it gives you:**
- Latest FastAPI patterns (`Annotated`-first, no deprecated `...` syntax)
- Dependency injection: `Depends()`, `yield`-based cleanup, class dependencies
- Returning Pydantic models for automatic validation, serialization, and OpenAPI docs
- Router organization: prefix/tags at router level, shared dependencies
- **Streaming endpoints**: SSE (`EventSourceResponse`), JSON Lines, byte streaming — essential for streaming LangGraph agent responses to the frontend
- Async vs. sync rules (and why mixing them destroys performance)
- Toolchain: `uv`, `Ruff`, `ty`, `SQLModel`, `HTTPX`, `Asyncer`

---

### 🛠️ `skill-creator` — The Meta-Layer
> *"Build the tools that build your tools."*

A complete framework for creating, evaluating, and improving new skills in this very format. Includes agent-based skill grading, benchmark generation, and A/B evaluation pipelines.

**Use it when**: You've identified a domain not covered by existing skills and want to formalize your knowledge in the same structured, evaluatable format.

---

## ⚡ What You Can Build

With this complete skill stack, an AI coding assistant can generate production-ready systems like these in a single session:

| System | Skills Used |
|---|---|
| **Autonomous research agent** with web search, memory, and email approval | `langchain-langgraph` + `prompt-engineering` + `cost-optimization` |
| **Document Q&A chatbot** over proprietary PDFs with source citations | `rag-implementation` + `langchain-langgraph` + `prompt-engineering` |
| **Multi-agent content pipeline** (researcher → writer → reviewer) | `langchain-langgraph` + `tool-design` + `cost-optimization` |
| **Structured data extraction API** from unstructured documents | `instructor` + `fastapi` + `prompt-engineering` |
| **Agent with MCP-powered database tools** | `mcp-builder` + `langchain-langgraph` + `tool-design` |
| **Cost-aware customer support agent** with long-term user memory | All skills combined |

---

## 🏁 Getting Started

### Prerequisites

```bash
pip install langgraph langchain langchain-anthropic langchain-openai
pip install langgraph-checkpoint-postgres  # for production memory
pip install instructor fastapi uvicorn
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

These skills are designed to be read by AI coding assistants that support the **skills protocol** (e.g., [Antigravity](https://antigravity.dev), Claude with project files).

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
    ├── langchain-langgraph/          # 🏗️ Core agent framework
    │   ├── SKILL.md                  #    Main skill (18KB)
    │   └── references/
    │       ├── architecture_patterns.md   # 13 patterns (22KB)
    │       └── advanced_features.md       # Advanced topics (20KB)
    │
    ├── prompt-engineering/           # 🎯 Prompt quality
    │   └── SKILL.md                  #    10 sections (24KB)
    │
    ├── tool-design/                  # 🔧 Tool contracts
    │   └── SKILL.md
    │
    ├── rag-implementation/           # 📚 Knowledge retrieval
    │   └── SKILL.md                  #    5 patterns + 4 vector stores
    │
    ├── instructor/                   # 📊 Structured outputs
    │   ├── SKILL.md
    │   └── references/               #    Validation, providers, examples
    │
    ├── cost-optimization/            # 💰 Cost control
    │   └── SKILL.md                  #    7 strategies + playbook
    │
    ├── mcp-builder/                  # 🔌 MCP servers
    │   ├── SKILL.md
    │   ├── reference/                #    Best practices + SDK guides
    │   └── scripts/                  #    Evaluation scripts
    │
    ├── fastapi/                      # 🚀 API serving
    │   ├── SKILL.md
    │   └── references/               #    Streaming, deps, tools
    │
    └── skill-creator/                # 🛠️ Meta-tooling
        ├── SKILL.md
        ├── agents/                   #    Analyzer, grader, comparator
        ├── scripts/                  #    Evaluation pipeline
        └── eval-viewer/              #    HTML review UI
```

---

## 🔗 Related Projects & Resources

> Discovered this repo while searching for one of these? You're in the right place.

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
