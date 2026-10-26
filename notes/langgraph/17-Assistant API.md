# Assistant API

可以通过`Agent Server`暴露的`Assistant API`管理AI助手

API文档地址：http://localhost:2024/docs#tag/assistants

API调用：推荐 `langgraph_sdk`

## 安装 LangGraph SDK

本课件使用 `langgraph-sdk` 访问 Assistant API，先通过 UV 安装：

```python
!uv add langgraph-sdk==0.4.2
```

## 创建客户端

连接本地 Agent Server（默认端口 2024），获得 `client.assistants` 子客户端：

```python
from langgraph_sdk import get_client

# 连接本地 Agent Server
client = get_client(url="http://localhost:2024")
client
```

## 创建 Assistant

> POST /assistants

创建时会为该 Assistant 生成初始版本（version=1）。

```python
# 为 agent 图创建 Assistant
# - context：静态上下文，会注入到图的 Runtime 中
# - metadata：附加元数据，可用于后续搜索过滤
assistant = await client.assistants.create(
    graph_id="agent",
    name="假模型", # 取一个直观的名称，用户友好，无功能含义
    description="最多允许运行10个超步", # 描述，无功能含义
    context={"model": "fake"},
    config={
        "recursion_limit": 10
    }
)
assistant
```

保存 Assistant ID 供后续接口使用：

```python
assistant_id = assistant["assistant_id"]
assistant_id
```

## 获取 Assistant

> GET /assistants/{assistant_id}

通过 ID 获取 Assistant 的详细信息：

```python
assistant = await client.assistants.get(assistant_id)
assistant
```

## 更新 Assistant

> PATCH /assistants/{assistant_id}

更新 Assistant 的名称、描述、配置等。更新后会生成一个新版本，并将其设为最新版本。

```python
assistant_v2 = await client.assistants.update(
    assistant_id,
    description="最多允许20个超步",
    config={
        "recursion_limit":20
    }
)
assistant_v2
```

## 获取 Assistant 版本列表

> POST /assistants/{assistant_id}/versions

列出 Assistant 的全部历史版本：

```python
versions = await client.assistants.get_versions(assistant_id)
versions
```

## 设置最新版本

> POST /assistants/{assistant_id}/latest

将某个历史版本设为最新版本，用于版本回滚：

```python
# 回滚到版本 1
assistant = await client.assistants.set_latest(assistant_id, version=1)
print(f"当前最新版本: {assistant['version']}, 名称: {assistant['name']}")
```

## 搜索 Assistant

> POST /assistants/search

按名称、graph_id、metadata 过滤，支持分页与排序。该接口同样用于列出全部 Assistant。

```python
# 按名称模糊搜索（不区分大小写的子串匹配）
result = await client.assistants.search(
    graph_id="agent",
    limit=10,
)
result
```

使用 `select` 可指定返回字段：

```python
result = await client.assistants.search(
    limit=2,
    offset=0,
    sort_by="created_at",
    sort_order="desc",
    select=["assistant_id", "name", "created_at"]
)
result
```

## 统计 Assistant 数量

> POST /assistants/count

按条件统计 Assistant 数量：

```python
count = await client.assistants.count()
print(f"数量: {count}")
```

## 获取 Assistant 图结构

> GET /assistants/{assistant_id}/graph

获取 Assistant 对应的图结构（节点、边）。

```python
graph = await client.assistants.get_graph(assistant_id)
print("节点：", [n for n in graph["nodes"]])
print("边：", [e for e in graph["edges"]])
```

## 获取 Assistant Schema

> GET /assistants/{assistant_id}/schemas

获取图输入、输出、状态、配置和上下文的 JSON Schema：

```python
schema = await client.assistants.get_schemas(assistant_id)
schema
```

## 删除 Assistant

> DELETE /assistants/{assistant_id}

删除 Assistant，其所有版本也会一并删除。删除后再次查询会抛出 `NotFoundError`：

```python
from langgraph_sdk.errors import NotFoundError

await client.assistants.delete(assistant_id)

try:
    await client.assistants.get(assistant_id)
except NotFoundError as e:
    print("已删除：", e)
```
