# langchain-dependencies

> 设置新项目、询问包版本、安装问题或 LangChain/LangGraph/LangSmith/Deep Agents 的依赖管理时参阅此文件。涵盖必需包、最低版本、环境要求、版本最佳实践，以及 Python 和 TypeScript 的常用社区工具包。

## 目录
- [环境要求](#环境要求)
- [框架选择](#框架选择)
- [核心包 — Python](#核心包--python)
- [核心包 — TypeScript](#核心包--typescript)
- [最简项目模板](#最简项目模板)
- [版本策略与升级](#版本策略与升级)
- [环境变量](#环境变量)
- [常见错误与修复](#常见错误与修复)

---

## 环境要求

| 要求 | Python | TypeScript / Node |
|------|--------|-------------------|
| 最低运行时 | **Python 3.10+** | **Node.js 20+** |
| LangChain | **1.0+ (LTS)** | **1.0+ (LTS)** |
| LangSmith SDK | >= 0.3.0 | >= 0.3.0 |

---

## 框架选择

选择**一个** Agent 编排层，不需要两者都用。

| 框架 | 使用时机 | 核心额外包 |
|------|---------|-----------|
| **LangGraph** | 需要细粒度图控制、自定义工作流、循环或分支 | `langgraph` / `@langchain/langgraph` |
| **Deep Agents** | 需要开箱即用的规划、记忆、文件上下文和技能 | `deepagents`（依赖 LangGraph；作为传递依赖安装） |

两者都建立在 `langchain` + `langchain-core` + `langsmith` 之上。

---

## 核心包 — Python

### Python — 始终必需

| 包 | 作用 | 最低版本 |
|----|------|---------|
| `langchain` | Agent、链、检索 | 1.0 |
| `langchain-core` | 基础类型和接口（对等依赖） | 1.0 |
| `langsmith` | 追踪、评估、数据集 | 0.3.0 |

### Python — 编排层（选一）

| 包 | 使用时机 | 最低版本 |
|----|---------|---------|
| `langgraph` | 直接构建自定义图 | 1.0 |
| `deepagents` | 使用 Deep Agents 框架 | latest |

### Python — 模型提供商（按需选择）

| 包 | 提供商 |
|----|--------|
| `langchain-openai` | OpenAI (GPT-4o, o3, …) |
| `langchain-anthropic` | Anthropic (Claude) |
| `langchain-google-genai` | Google (Gemini) |
| `langchain-mistralai` | Mistral |
| `langchain-groq` | Groq（快速推理） |
| `langchain-cohere` | Cohere |
| `langchain-fireworks` | Fireworks AI |
| `langchain-together` | Together AI |
| `langchain-huggingface` | Hugging Face Hub |
| `langchain-ollama` | Ollama（本地模型） |
| `langchain-aws` | AWS Bedrock |
| `langchain-azure-ai` | Azure AI Foundry |

### Python — 常用工具和检索包

| 包 | 增加功能 | 说明 |
|----|---------|------|
| `langchain-tavily` | Tavily 网络搜索 | 首选最新版 |
| `langchain-text-splitters` | 文本分块工具 | 语义化版本，保持最新 |
| `langchain-community` | 1000+ 集成（备用） | **非语义化版本——固定到 minor 系列** |
| `faiss-cpu` | FAISS 向量存储（本地） | 通过 langchain-community |
| `langchain-chroma` | Chroma 向量存储 | 首选最新版 |
| `langchain-pinecone` | Pinecone 向量存储 | 首选最新版 |
| `langchain-qdrant` | Qdrant 向量存储 | 首选最新版 |
| `langchain-weaviate` | Weaviate 向量存储 | 首选最新版 |
| `langsmith[pytest]` | pytest 插件 | 需要 langsmith >= 0.3.4 |

> ⚠️ **langchain-community 稳定性说明**：该包**未遵循语义化版本**。Minor 版本可能包含破坏性变更。当存在专用集成包时（如 `langchain-chroma` 代替社区的 Chroma 集成），优先使用——专用包独立版本控制且测试更全面。

---

## 核心包 — TypeScript

### TypeScript — 始终必需

| 包 | 作用 | 最低版本 |
|----|------|---------|
| `@langchain/core` | 基础类型和接口（对等依赖） | 1.0 |
| `langchain` | Agent、链、检索 | 1.0 |
| `langsmith` | 追踪、评估、数据集 | 0.3.0 |

### TypeScript — 编排层（选一）

| 包 | 使用时机 | 最低版本 |
|----|---------|---------|
| `@langchain/langgraph` | 直接构建自定义图 | 1.0 |
| `deepagents` | 使用 Deep Agents 框架 | latest |

### TypeScript — 模型提供商（按需选择）

| 包 | 提供商 |
|----|--------|
| `@langchain/openai` | OpenAI |
| `@langchain/anthropic` | Anthropic (Claude) |
| `@langchain/google-genai` | Google (Gemini) |
| `@langchain/mistralai` | Mistral |
| `@langchain/groq` | Groq |
| `@langchain/cohere` | Cohere |
| `@langchain/aws` | AWS Bedrock |
| `@langchain/ollama` | Ollama |

### TypeScript — 常用工具和检索包

| 包 | 增加功能 |
|----|---------|
| `@langchain/tavily` | Tavily 网络搜索 |
| `@langchain/community` | 广泛集成（谨慎使用，优先专用包） |
| `@langchain/pinecone` | Pinecone 向量存储 |
| `@langchain/qdrant` | Qdrant 向量存储 |

> ⚠️ **`@langchain/core` 必须显式安装** — 在 yarn workspaces 和 monorepos 中它是对等依赖，不会总是自动提升。

---

## 最简项目模板

### LangGraph — Python

```python
# requirements.txt
langchain>=1.0,<2.0
langchain-core>=1.0,<2.0
langgraph>=1.0,<2.0
langsmith>=0.3.0

# 添加你的模型提供商，例如：
# langchain-openai
# langchain-anthropic
```

### LangGraph — TypeScript

```json
{
  "dependencies": {
    "@langchain/core": "^1.0.0",
    "langchain": "^1.0.0",
    "@langchain/langgraph": "^1.0.0",
    "langsmith": "^0.3.0"
  }
}
```

### Deep Agents — Python

```python
# requirements.txt
deepagents            # 内部捆绑 langgraph
langchain>=1.0,<2.0
langchain-core>=1.0,<2.0
langsmith>=0.3.0

# 添加你的模型提供商：
# langchain-anthropic
# langchain-openai
```

### 带 Tavily 和向量存储的 LangGraph — Python

```python
# requirements.txt
langchain>=1.0,<2.0
langchain-core>=1.0,<2.0
langgraph>=1.0,<2.0
langsmith>=0.3.0

# 网络搜索
langchain-tavily          # 使用最新版；伙伴包，语义化版本

# 向量存储 — 选一：
langchain-chroma          # 使用最新版
# langchain-pinecone      # 使用最新版
# langchain-qdrant        # 使用最新版

langchain-text-splitters  # 使用最新版
# 你的模型提供商：
# langchain-openai / langchain-anthropic / 其他
```

---

## 版本策略与升级

| 包组 | 版本方案 | 安全升级策略 |
|------|---------|------------|
| `langchain`, `langchain-core` | 严格语义化版本（1.0 LTS） | 允许 minor：`>=1.0,<2.0` |
| `langgraph` / `@langchain/langgraph` | 严格语义化版本（v1 LTS） | 允许 minor：`>=1.0,<2.0` |
| `langsmith` | 严格语义化版本 | 允许 minor：`>=0.3.0` |
| 专用集成包（如 `langchain-tavily`） | 独立版本控制 | 允许 minor 更新；使用最新版 |
| `langchain-community` | **非语义化版本** | 固定 minor：`>=0.4.0,<0.5.0` |
| `deepagents` | 跟随项目发布 | 生产中固定到已测试版本 |

---

## 环境变量

```bash
# LangSmith（始终推荐用于可观测性）
LANGSMITH_API_KEY=<your-key>
LANGSMITH_PROJECT=<project-name>   # 可选，默认 "default"

# 模型提供商 — 设置你使用的那个
OPENAI_API_KEY=<your-key>
ANTHROPIC_API_KEY=<your-key>
GOOGLE_API_KEY=<your-key>
MISTRAL_API_KEY=<your-key>
GROQ_API_KEY=<your-key>

# 常用工具/检索服务
TAVILY_API_KEY=<your-key>
PINECONE_API_KEY=<your-key>
```

---

## 常见错误与修复

### ❌ 使用 LangChain 0.3（旧版）
```python
# 错误：遗留版本，无新功能，仅安全补丁
langchain>=0.3,<0.4

# 正确：LangChain 1.0 LTS
langchain>=1.0,<2.0
```

### ❌ langchain-community 未固定版本
```python
# 错误：允许可能破坏性的 minor 版本更新
langchain-community>=0.4

# 正确：固定到确切的 minor 系列
langchain-community>=0.4.0,<0.5.0
```

### ❌ 使用已废弃的社区导入路径
```python
# 错误 — 已废弃的社区导入路径
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_community.vectorstores import Chroma
from langchain_community.vectorstores import Pinecone

# 正确 — 使用专用包导入
from langchain_tavily import TavilySearch            # pip: langchain-tavily
from langchain_chroma import Chroma                  # pip: langchain-chroma
from langchain_pinecone import PineconeVectorStore   # pip: langchain-pinecone
```

### ❌ TypeScript 中缺少 @langchain/core
```json
// 错误：在 yarn workspaces 中可能无法正确提升
{ "dependencies": { "@langchain/langgraph": "^1.0.0" } }

// 正确：始终显式列出 @langchain/core
{ "dependencies": { "@langchain/core": "^1.0.0", "@langchain/langgraph": "^1.0.0" } }
```

### ❌ Python 版本过低
```python
# 安装前验证
import sys
assert sys.version_info >= (3, 10), "LangChain 1.0 需要 Python 3.10+"
```
