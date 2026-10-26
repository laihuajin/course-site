# Milvus DML

```python
from pymilvus import AsyncMilvusClient

collection_name = "feishu_doc"
client = AsyncMilvusClient(
    uri="http://127.0.0.1:19530",
    db_name="deal_pilot"
)
```

## 准备数据

```python
import pypdf

pdf_path = "../resource/ECMA.pdf"

reader = pypdf.PdfReader(pdf_path)

chunks: list[str] = []

for page in reader.pages:
    chunks.append(page.extract_text())

len(chunks)
```

```python
from deal_pilot.rag.embedding_model import create_embedding_model

vectors = await create_embedding_model().aembed_documents(chunks)
len(vectors)
```

```python
entities = [
    {"text":text, "vector": vector} for text, vector in zip(chunks, vectors)
]
```

```python
len(entities)
```

```python
entities[100]
```

## 增

```python
# 一次性批量写入
await client.insert(
    collection_name=collection_name,
    data=entities
)
```

## 删

```python
await client.delete(
    collection_name=collection_name,
    ids=[469887312720117739]
)
```

# 改

```python
# upsert 可增可改，主键不存在时
await client.upsert(
    collection_name=collection_name,
    data={
        "id": 469887312720117740,
        "text": "目录"
    },
    partial_update=True # 只修改提供的字段
)
```
