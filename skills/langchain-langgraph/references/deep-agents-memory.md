# deep-agents-memory

> Deep Agent 需要记忆、持久化或文件系统访问时参阅此文件。涵盖 StateBackend（临时）、StoreBackend（持久）、FilesystemMiddleware，以及用于路由的 CompositeBackend。

## Backend 选择

| 使用场景 | Backend | 理由 |
|---------|---------|------|
| 临时工作文件 | StateBackend | 默认，无需配置 |
| 本地开发 CLI | FilesystemBackend | 直接磁盘访问 |
| 跨会话记忆 | StoreBackend | 跨线程持久 |
| 混合存储 | CompositeBackend | 混合临时 + 持久 |

`FilesystemMiddleware` 提供工具：`ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`

---

## 默认 StateBackend（临时）

```python
from deepagents import create_deep_agent

agent = create_deep_agent()  # 默认：StateBackend
result = agent.invoke({
    "messages": [{"role": "user", "content": "Write notes to /draft.txt"}]
}, config={"configurable": {"thread_id": "thread-1"}})
# /draft.txt 在线程结束时丢失
```

---

## CompositeBackend（混合存储）

按路径前缀将文件路由到不同 Backend：

```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

composite_backend = lambda rt: CompositeBackend(
    default=StateBackend(rt),
    routes={"/memories/": StoreBackend(rt)}
)

agent = create_deep_agent(backend=composite_backend, store=store)

# /draft.txt -> 临时 (StateBackend)
# /memories/user-prefs.txt -> 持久 (StoreBackend)
```

---

## 跨会话记忆

```python
# /memories/ 中的文件通过 StoreBackend 跨线程持久
config1 = {"configurable": {"thread_id": "thread-1"}}
agent.invoke({"messages": [{"role": "user", "content": "Save to /memories/style.txt"}]}, config=config1)

config2 = {"configurable": {"thread_id": "thread-2"}}
agent.invoke({"messages": [{"role": "user", "content": "Read /memories/style.txt"}]}, config=config2)
# 线程 2 可以读取线程 1 保存的文件
```

---

## FilesystemBackend（本地开发）

```python
from deepagents.backends import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

agent = create_deep_agent(
    backend=FilesystemBackend(root_dir=".", virtual_mode=True),  # 限制访问
    interrupt_on={"write_file": True, "edit_file": True},
    checkpointer=MemorySaver()
)
# Agent 可以读写磁盘上的实际文件
```

**⚠️ 安全警告：永远不要在 Web 服务器中使用 FilesystemBackend — 改用 StateBackend 或沙箱。**

---

## 在自定义工具中使用 Store

```python
from langchain.tools import tool, ToolRuntime
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore

@tool
def get_user_preference(key: str, runtime: ToolRuntime) -> str:
    """Get a user preference from long-term storage."""
    store = runtime.store
    result = store.get(("user_prefs",), key)
    return str(result.value) if result else "Not found"

@tool
def save_user_preference(key: str, value: str, runtime: ToolRuntime) -> str:
    """Save a user preference to long-term storage."""
    store = runtime.store
    store.put(("user_prefs",), key, {"value": value})
    return f"Saved {key}={value}"

store = InMemoryStore()
agent = create_agent(model="gpt-4.1", tools=[get_user_preference, save_user_preference], store=store)
```

---

## 常见错误与修复

### ❌ StoreBackend 缺少 store 实例
```python
# 错误
agent = create_deep_agent(backend=lambda rt: StoreBackend(rt))

# 正确
agent = create_deep_agent(backend=lambda rt: StoreBackend(rt), store=InMemoryStore())
```

### ❌ StateBackend 文件不跨线程持久
```python
# 错误：thread-2 无法读取 thread-1 的文件
agent.invoke({...}, config={"configurable": {"thread_id": "thread-1"}})  # 写入
agent.invoke({...}, config={"configurable": {"thread_id": "thread-2"}})  # 找不到文件！

# 正确：使用 CompositeBackend 与 StoreBackend 路由
```

### ❌ 路径前缀不匹配
```python
# routes={"/memories/": StoreBackend(rt)} 的情况下：
agent.invoke(...)  # /prefs.txt -> 临时（不匹配）
agent.invoke(...)  # /memories/prefs.txt -> 持久（匹配路由）
```

### ❌ 生产环境使用 InMemoryStore
```python
# 错误：重启后数据丢失
store = InMemoryStore()

# 正确：生产使用 PostgresStore
store = PostgresStore(connection_string="postgresql://...")
```

### ❌ FilesystemBackend 缺少 virtual_mode
```python
# 正确：启用 virtual_mode=True 防止 ../ 和 ~/ 路径逃逸
backend = FilesystemBackend(root_dir="/project", virtual_mode=True)
```

> **CompositeBackend 最长前缀优先匹配：**
> `routes={"/mem/": StoreBackend(rt), "/mem/temp/": StateBackend(rt)}`
> - `/mem/file.txt` → StoreBackend；`/mem/temp/file.txt` → StateBackend（更长的前缀匹配）
