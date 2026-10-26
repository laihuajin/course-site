# Store

`Checkpointer` 保存的是某个 `thread` 的图状态，不同线程之间的状态彼此隔离。但是在真实应用中，我们往往还需要保存一些**跨线程、跨会话的长期数据**。

`Store` 就是用来存储这类数据的。它不依附于某一次图的执行，可以让 Agent 在新的对话中仍然记得用户，也可以在多个线程或多个 Agent 之间共享数据。

通常可以用于实现长期记忆：

- **用户偏好**：用户曾经说过“回答时请使用中文”。即使下次开启了新对话，Agent 仍然可以从 `Store` 中读取这个偏好。
- **用户资料**：记住用户的姓名、职业、所在城市等稳定信息，在之后的不同会话中提供个性化服务。
- **状态跟踪**：把“用户正在学习 LangGraph”这类值得长期保留的信息从对话状态中提取出来，写入 `Store`。
- ...

## LangGraph API

### 创建 Memory Store

Store 中的一条数据由三部分组成：

- `namespace`：命名空间，用元组表示，可以理解为数据所在的目录
- `key`：数据在当前命名空间中的唯一标识
- `value`：真正保存的数据，必须是字典

```
Store
└── namespace（命名空间）
    ├── key → value
    ├── key → value
    └── key → value
```

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
namespace = ("users", "user_001", "memory")
```

### 增加数据

使用异步方法 `aput` 写入数据。同一个命名空间中，`key` 唯一标识一条数据：

```python
await store.aput(
    namespace,
    key="profile",
    value={"name": "张三", "city": "上海", "job": "Python 工程师"},
)

await store.aput(
    namespace,
    key="preferences",
    value={"language": "中文", "response_style": "简洁"},
)
```

### 查询单条数据

使用异步方法 `aget` 根据 `namespace + key` 精确查询一条数据：

```python
profile = await store.aget(namespace, key="profile")
profile
```

### 查询多条数据

使用异步方法 `asearch` 查询某个命名空间下的多条数据：

```python
items = await store.asearch(namespace)
[(item.key, item.value) for item in items]
```

### 查询子命名空间

使用异步方法 `alist_namespaces` 查询父命名空间下的子命名空间。`prefix` 指定父命名空间，`max_depth` 限制返回的命名空间深度：

```python
namespaces = await store.alist_namespaces(
    prefix=("users",),
    max_depth=4,
)
namespaces
```

### 修改数据

Store 没有单独的 `update` 方法。使用相同的 `namespace + key` 再次调用 `aput`，就会覆盖原数据：

```python
await store.aput(
    namespace,
    key="profile",
    value={"name": "张三", "city": "杭州", "job": "AI 工程师"},
)

updated_profile = await store.aget(namespace, key="profile")
updated_profile
```

`aput` 会整体替换原来的 `value`，不会自动合并新旧字典。

### 删除数据

使用异步方法 `adelete` 删除指定数据。删除不存在的数据不会报错：

```python
await store.adelete(namespace, key="profile")

# 删除后查询不到，返回 None
await store.aget(namespace, key="profile")
```

最后清理示例中剩余的数据：

```python
await store.adelete(namespace, key="preferences")
await store.asearch(namespace)
```

## Agent Server API

Agent Server 通过 Store API 对外提供长期存储能力。这些数据不依赖图、Assistant 或 Thread，可以直接通过 `client.store` 访问。

API 文档地址：http://localhost:2024/docs#tag/store

### 创建客户端

连接本地 Agent Server（默认端口 2024）：

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:2024")
server_namespace = ("users", "user_002", "memory")
```

### 增加数据

> PUT /store/items

使用 `put_item` 写入数据：

```python
await client.store.put_item(
    server_namespace,
    key="profile",
    value={"name": "李四", "city": "北京", "job": "Python 工程师"},
)

await client.store.put_item(
    server_namespace,
    key="preferences",
    value={"language": "中文", "response_style": "简洁"},
)
```

### 查询单条数据

> GET /store/items

使用 `get_item` 根据 `namespace + key` 精确查询一条数据：

```python
profile = await client.store.get_item(
    server_namespace,
    key="profile",
)
profile
```

### 查询多条数据

> POST /store/items/search

使用 `search_items` 查询某个命名空间下的多条数据：

```python
result = await client.store.search_items(server_namespace)
[(item["key"], item["value"]) for item in result["items"]]
```

### 查询子命名空间

> POST /store/namespaces

使用 `list_namespaces` 查询父命名空间下的子命名空间：

```python
result = await client.store.list_namespaces(
    prefix=["users"],
    max_depth=4,
)
result["namespaces"]
```

### 修改数据

> PUT /store/items

Store API 同样没有单独的更新接口。使用相同的 `namespace + key` 再次调用 `put_item`，就会覆盖原数据：

```python
await client.store.put_item(
    server_namespace,
    key="profile",
    value={"name": "李四", "city": "深圳", "job": "AI 工程师"},
)

updated_profile = await client.store.get_item(
    server_namespace,
    key="profile",
)
updated_profile
```

`put_item` 会整体替换原来的 `value`，不会自动合并新旧字典。

### 删除数据

> DELETE /store/items

使用 `delete_item` 删除指定数据：

```python
await client.store.delete_item(server_namespace, key="profile")

# 通过搜索确认 profile 已被删除
result = await client.store.search_items(server_namespace)
[(item["key"], item["value"]) for item in result["items"]]
```

最后清理示例中剩余的数据：

```python
await client.store.delete_item(server_namespace, key="preferences")
await client.store.search_items(server_namespace)
```
