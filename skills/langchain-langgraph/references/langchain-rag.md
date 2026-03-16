# langchain-rag

> 构建**任何** RAG（检索增强生成）系统时参阅此文件。涵盖文档加载器、RecursiveCharacterTextSplitter、嵌入模型（OpenAI），以及向量存储（Chroma, FAISS, Pinecone）。

## 目录
- [RAG 管道概述](#rag-管道概述)
- [向量存储选择](#向量存储选择)
- [完整 RAG 管道示例](#完整-rag-管道示例)
- [文档加载器](#文档加载器)
- [文本分割](#文本分割)
- [向量存储操作](#向量存储操作)
- [检索策略](#检索策略)
- [RAG + Agent 集成](#rag--agent-集成)
- [常见错误与修复](#常见错误与修复)

---

## RAG 管道概述

**处理流程：**
1. **索引**：加载 → 分割 → 嵌入 → 存储
2. **检索**：查询 → 嵌入 → 搜索 → 返回文档
3. **生成**：文档 + 查询 → LLM → 响应

**关键组件：**
- **文档加载器**：从文件、网络、数据库摄取数据
- **文本分割器**：将文档分割成块
- **嵌入模型**：将文本转换为向量
- **向量存储**：存储和搜索嵌入

---

## 向量存储选择

| 向量存储 | 使用场景 | 持久性 |
|---------|---------|--------|
| **InMemory** | 测试 | 仅内存 |
| **FAISS** | 本地，高性能 | 磁盘 |
| **Chroma** | 开发 | 磁盘 |
| **Pinecone** | 生产，托管 | 云端 |

---

## 完整 RAG 管道示例

**Python：**
```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import InMemoryVectorStore
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

# 1. 加载文档
docs = [
    Document(page_content="LangChain is a framework for LLM apps.", metadata={}),
    Document(page_content="RAG = Retrieval Augmented Generation.", metadata={}),
]

# 2. 分割文档
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
splits = splitter.split_documents(docs)

# 3. 创建嵌入和存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = InMemoryVectorStore.from_documents(splits, embeddings)

# 4. 创建检索器
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# 5. 在 RAG 中使用
model = ChatOpenAI(model="gpt-4.1")
query = "What is RAG?"
relevant_docs = retriever.invoke(query)

context = "\n\n".join([doc.page_content for doc in relevant_docs])
response = model.invoke([
    {"role": "system", "content": f"Use this context:\n\n{context}"},
    {"role": "user", "content": query},
])
```

**TypeScript：**
```typescript
import { ChatOpenAI, OpenAIEmbeddings } from "@langchain/openai";
import { MemoryVectorStore } from "@langchain/classic/vectorstores/memory";
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";
import { Document } from "@langchain/core/documents";

// 1. 加载文档
const docs = [
  new Document({ pageContent: "LangChain is a framework for LLM apps.", metadata: {} }),
  new Document({ pageContent: "RAG = Retrieval Augmented Generation.", metadata: {} }),
];

// 2. 分割文档
const splitter = new RecursiveCharacterTextSplitter({ chunkSize: 500, chunkOverlap: 50 });
const splits = await splitter.splitDocuments(docs);

// 3. 创建嵌入和存储
const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const vectorstore = await MemoryVectorStore.fromDocuments(splits, embeddings);

// 4. 创建检索器
const retriever = vectorstore.asRetriever({ k: 4 });

// 5. 在 RAG 中使用
const model = new ChatOpenAI({ model: "gpt-4.1" });
const query = "What is RAG?";
const relevantDocs = await retriever.invoke(query);

const context = relevantDocs.map(doc => doc.pageContent).join("\n\n");
const response = await model.invoke([
  { role: "system", content: `Use this context:\n\n${context}` },
  { role: "user", content: query },
]);
```

---

## 文档加载器

### 加载 PDF
```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("./document.pdf")
docs = loader.load()
print(f"Loaded {len(docs)} pages")
```

### 加载网页
```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://docs.langchain.com")
docs = loader.load()
```

### 加载目录
```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader

loader = DirectoryLoader(
    "path/to/documents",
    glob="**/*.txt",  # 文件模式
    loader_cls=TextLoader
)
docs = loader.load()
```

---

## 文本分割

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,        # 每块的字符数
    chunk_overlap=200,      # 上下文连续性的重叠
    separators=["\n\n", "\n", " ", ""],  # 分割层级
)

splits = splitter.split_documents(docs)
```

---

## 向量存储操作

### Chroma（持久化本地存储）

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

# 创建并保存
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=OpenAIEmbeddings(),
    persist_directory="./chroma_db",
    collection_name="my-collection",
)

# 从磁盘加载
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=OpenAIEmbeddings(),
    collection_name="my-collection",
)
```

### FAISS（高性能本地存储）

```python
from langchain_community.vectorstores import FAISS

vectorstore = FAISS.from_documents(splits, embeddings)
vectorstore.save_local("./faiss_index")

# 加载（需要 allow_dangerous_deserialization）
loaded = FAISS.load_local(
    "./faiss_index",
    embeddings,
    allow_dangerous_deserialization=True
)
```

---

## 检索策略

### 相似度搜索

```python
# 基础搜索
results = vectorstore.similarity_search(query, k=5)

# 带分数
results_with_score = vectorstore.similarity_search_with_score(query, k=5)
for doc, score in results_with_score:
    print(f"Score: {score}, Content: {doc.page_content}")
```

### MMR 搜索（平衡相关性和多样性）

```python
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={"fetch_k": 20, "lambda_mult": 0.5, "k": 5},
)
```

### 元数据过滤

```python
docs = [
    Document(
        page_content="Python programming guide",
        metadata={"language": "python", "topic": "programming"}
    ),
]

# 按元数据过滤搜索
results = vectorstore.similarity_search(
    "programming",
    k=5,
    filter={"language": "python"}  # 只返回 Python 文档
)
```

---

## RAG + Agent 集成

将 RAG 作为 Agent 的工具：

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def search_docs(query: str) -> str:
    """Search documentation for relevant information."""
    docs = retriever.invoke(query)
    return "\n\n".join([d.page_content for d in docs])

agent = create_agent(
    model="gpt-4.1",
    tools=[search_docs],
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "How do I create an agent?"}]
})
```

---

## 常见错误与修复

### ❌ Chunk Size 过小或过大
```python
# 错误：太小（丢失上下文）或太大（超出限制）
splitter = RecursiveCharacterTextSplitter(chunk_size=50)
splitter = RecursiveCharacterTextSplitter(chunk_size=10000)

# 正确：500-1500 通常适合
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
```

### ❌ 没有 Chunk Overlap
```python
# 错误：没有重叠 - 边界处上下文中断
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=0)

# 正确：10-20% 重叠
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
```

### ❌ 使用 InMemory 向量存储用于生产
```python
# 错误：重启后丢失
vectorstore = InMemoryVectorStore.from_documents(docs, embeddings)

# 正确
vectorstore = Chroma.from_documents(docs, embeddings, persist_directory="./chroma_db")
```

### ❌ 索引和查询使用不同的嵌入模型
```python
# 错误：不兼容！
vectorstore = Chroma.from_documents(docs, OpenAIEmbeddings(model="text-embedding-3-small"))
retriever = vectorstore.as_retriever(embeddings=OpenAIEmbeddings(model="text-embedding-3-large"))

# 正确：同一模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(docs, embeddings)
retriever = vectorstore.as_retriever()  # 使用相同嵌入
```

### ❌ 加载 FAISS 时忘记反序列化参数
```python
# 错误：将引发错误
loaded_store = FAISS.load_local("./faiss_index", embeddings)

# 正确
loaded_store = FAISS.load_local("./faiss_index", embeddings, allow_dangerous_deserialization=True)
```

### ❌ 嵌入维度不匹配
```python
# 错误：索引有 1536 维，但使用 512 维嵌入
pc.create_index(name="idx", dimension=1536, metric="cosine")
vectorstore = PineconeVectorStore.from_documents(
    docs, OpenAIEmbeddings(model="text-embedding-3-small", dimensions=512), index=pc.Index("idx")
)  # 错误：维度不匹配！

# 正确：匹配维度
embeddings = OpenAIEmbeddings()  # 默认 1536
```
