# Milvus DQL

| 方法 | 输入 | 用途 | 返回结果 |
|---|---|---|---|
| `query()` | 主键或标量过滤条件 | 按 ID 查询、筛选、统计 | 实体列表或统计结果 |
| `search()` | 查询向量，可附带过滤条件 | 按相似度查 Top-K | 每个查询向量对应一组结果 |

## 环境准备

```python
from pymilvus import AsyncMilvusClient

collection_name = "feishu_doc"
client = AsyncMilvusClient(
    uri="http://127.0.0.1:19530",
    db_name="deal_pilot",
)
```

查询前需要加载 Collection。加载让查询所需的数据和索引可用，不等于重新写入数据。

```python
await client.load_collection(collection_name=collection_name)
await client.get_load_state(collection_name=collection_name)
```

## 标量查询

```python
rows = await client.query(
    # 要查询的集合
    collection_name=collection_name,
    # 标量过滤条件：text字段包含生成器，
    filter='text like "%生成器%"',
    # 查询结果包含的字段
    output_fields=["id", "text"],
    # 查询多少条记录
    limit=5
)
from rich import print
for row in rows:
    print("id:", row["id"])
    print("text:", row["text"])
    print("-"*60)
```

## 向量检索

### 问题转向量

```python
from deal_pilot.rag.embedding_model import create_embedding_model

embedding_model = create_embedding_model()

question = "ES6中正则表达式有什么变动？"
query_vector = await embedding_model.aembed_query(question)

len(query_vector)
```

### Top-K 检索

`anns_field` 指定向量字段，`data` 是查询向量列表，`limit` 是每个查询最多返回的结果数。

```python
results = await client.search(
    collection_name=collection_name,
    data=[query_vector],
    anns_field="vector", # 若不指定，则按照 Schema 中唯一的向量字段查询
    limit=5,
    search_params={
        "metric_type": "COSINE",
        "params": {"ef": 64},
    },
    output_fields=["text"],
)
for rank, hit in enumerate(results[0], start=1):
    print(dict.keys(hit))
    print(f"第 {rank} 名，id={hit['id']}，COSINE={hit['distance']:.4f}")
    print(hit["entity"]["text"])
    print('-'*60)
```

```python
def show_result(results):
    for i, result in enumerate(results, start=1):
        print(f"问题{i}的检索结果：")
        for rank, hit in enumerate(result, start=1):
            print(dict.keys(hit))
            print(f"第 {rank} 名，id={hit['id']}，COSINE={hit['distance']:.4f}")
            print(hit["entity"]["text"])
            print('-'*60)
        print('='*60)
```

```python
show_result(results)
```

### 过滤条件

```python
filtered_results = await client.search(
    collection_name=collection_name,
    anns_field="vector",
    data=[query_vector],
    filter='text like "%正则%"',
    limit=5,
    output_fields=["text"],
)

show_result(filtered_results)
```

### 范围检索

Top-K 限制返回数量，范围检索进一步限制相似度。对于 COSINE，`radius` 是分数下界，返回分数需要大于它。

```python
range_results = await client.search(
    collection_name=collection_name,
    anns_field="vector",
    data=[query_vector],
    limit=5,
    search_params={
        "metric_type": "COSINE",
        "params": {"ef": 64, "radius": 0.8},
    },
    output_fields=["text"],
)

show_result(range_results)
```

## 关闭连接

```python
await client.close()
```
