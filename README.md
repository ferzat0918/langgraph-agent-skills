# LangGraph Agent Skills

> A curated collection of AI coding skills for building production-grade agents with LangGraph.

## 📦 Skills Included

| Skill | Description |
|---|---|
| `langchain-langgraph` | Core LangGraph agent patterns: StateGraph, HITL, Memory, Multi-Agent |
| `prompt-engineering` | System prompts, tool descriptions, CoT, LLM-as-Judge, A/B testing |
| `tool-design` | Tool contract design, consolidation principle, MCP naming |
| `rag-implementation` | RAG patterns: Hybrid Search, HyDE, Reranking, Vector Stores |
| `instructor` | Structured LLM outputs with Pydantic validation and auto-retry |
| `cost-optimization` | Model routing, Prompt Caching, Token budgets, Batch API |
| `mcp-builder` | MCP server development guide (Python/TypeScript) |
| `fastapi` | FastAPI best practices for serving agent APIs |
| `skill-creator` | Meta-skill for creating new skills |

## 🚀 Usage

These skills are designed to work with AI coding assistants (e.g., Antigravity/Claude).
Place the `skills/` directory where your assistant can read them.

## 📁 Structure

```
langgragh_skills/
└── skills/
    ├── langchain-langgraph/
    │   ├── SKILL.md
    │   └── references/
    │       ├── architecture_patterns.md
    │       └── advanced_features.md
    ├── prompt-engineering/
    │   └── SKILL.md
    ├── tool-design/
    │   └── SKILL.md
    ├── rag-implementation/
    │   └── SKILL.md
    ├── instructor/
    │   └── SKILL.md
    ├── cost-optimization/
    │   └── SKILL.md
    ├── mcp-builder/
    │   └── SKILL.md
    ├── fastapi/
    │   └── SKILL.md
    └── skill-creator/
        └── SKILL.md
```

## ⚠️ Security Note

Never commit `.env` files or any files containing API keys.
Use `.env.example` as a template for required environment variables.
