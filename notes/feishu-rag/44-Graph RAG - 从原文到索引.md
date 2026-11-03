# Graph RAG - 从原文到索引

原文 → 切片 → 实体与关系 → 社区 → 社区报告 → 向量

## 动机问题

> 《奥本海默》中饰演奥本海默的演员，在另一部电影里演了谁？

### 遇到了什么问题？

翻开《奥本海默》的资料，可以查到饰演者是**希里安·墨菲**，但这篇原文只介绍这一部电影。再翻开《盗梦空间》的资料，才会发现他还饰演了**罗伯特·费舍尔**。两篇原文各给出一半答案，连接它们的是同一位演员，而用户的问题里恰好没有写出演员姓名。

### Agentic RAG 能回答这个问题吗？

可以。第一轮用“谁饰演了奥本海默”检索，读到演员姓名；第二轮把刚得到的“希里安·墨菲”作为检索词，就有机会找到《盗梦空间》的原文，核实他饰演的角色。即使一次向量检索漏掉了第二篇，Agent 也可以根据第一轮结果补查。对于这三份资料，两轮文本检索就够了。

### Graph RAG 怎样把两份证据连起来？

建索引时，从两篇原文分别提取“谁参演了哪部电影、饰演了哪个角色”，把两处“希里安·墨菲”关联到同一个人物，并保留每条关系对应的原文切片。

![希里安·墨菲在《奥本海默》中饰演罗伯特·奥本海默，在《盗梦空间》中饰演罗伯特·费舍尔](assets/graphrag-movie-evidence.png)

回答时，先用《奥本海默》的原文确定演员，沿这个人物关联到《盗梦空间》，再用第二篇原文核实角色。Agent 也可以把图查询作为工具。它仍负责决定下一步查什么，只是多了一种利用已有关系查找证据的方式。

### 什么时候值得提前整理关系？

> 假设片库扩大到一万部电影。找出所有合作过至少三次的导演和演员，并列出每对搭档合作的影片。

这里最难的是“所有”。如果 Agent 每次只能搜到最相关的几段原文，它可以查到某对搭档合作过三次，却很难知道还有哪些搭档一次都没搜到。逐篇读完当然也能统计，但下次换个条件，又需要用到这些演职员关系。

假设建索引时已经把导演、演员和电影对应好，就可以沿着 **导演 → 执导的电影 ← 参演该片的演员** 找到每对搭档，再按搭档统计不同影片的数量。把门槛从三次改为五次，只需重新筛选。结果的完整性取决于索引有没有漏掉演职员或认错同名人物。这类精确统计用图查询或关系表都能完成。

### 为什么还要划分社区、生成报告？

> 纵观整个片库，有哪些经常一起工作的创作群体？各群体的作品有什么共同特点？

这个问题没有给出具体人名，还需要从作品资料中归纳特点。微软 GraphRAG 会在索引阶段把联系紧密的实体划成社区，再根据各社区的资料生成报告。这样，即使两个创作群体之间几乎没有交集，后续的 Global Search 也可以综合多个社区的报告，分别概括它们的特点。[微软 GraphRAG 的 Global Search 文档](https://microsoft.github.io/graphrag/query/global_search/)

下面仍用三部电影演示从原文到索引的过程。查看实体和关系时，可以留意同一位演员如何连接两部作品；读社区报告时，再看模型把哪些零散事实归纳到了一起。

## 安装依赖

```python
import sys

!uv pip install --python "{sys.executable}" "graphrag==3.3.0"
```

```python
from pathlib import Path

import graphrag.api as api
import pandas as pd
from graphrag.config.models.graph_rag_config import GraphRagConfig
from graphrag_llm.config import ModelConfig
```

## 填写索引配置

在下面的单元格直接填写 Chat 和 Embedding 的模型名、OpenAI 兼容接口地址和 API Key。两个服务可以分别配置。向量维度填写 Embedding 模型实际返回的长度。分享课件前，请把真实密钥换回占位符。

本课的工作目录是工程内的 `resource/graph_rag/`。各步骤共用下面配置的路径。

| 子目录 | 保存内容 |
| --- | --- |
| `input/` | 三份电影演职员资料，每部电影一份 UTF-8 文本 |
| `prompts/` | 电影索引使用的三个中文提示词模板 |
| `output/` | 各阶段的 Parquet 表、运行统计 `stats.json` 和上下文 `context.json` |
| `output/lancedb/` | 实体描述、社区报告和文本切片的向量表 |
| `cache/` | 可在重复构建索引时复用的模型调用结果 |
| `logs/` | 索引日志 `indexing-engine.log` |
| `update_output/` | 增量更新的产物，仅在执行增量索引时使用 |

```python
# 支持从工程根目录或课件目录启动 Notebook。
project_dir = Path.cwd()
if project_dir.name == "课件":
    project_dir = project_dir.parent
if not (project_dir / "langgraph.json").is_file():
    raise FileNotFoundError("请从工程根目录或课件目录启动 Notebook")

work_dir = project_dir / "resource" / "graph_rag"
input_dir = work_dir / "input"
output = work_dir / "output"
input_dir.mkdir(parents=True, exist_ok=True)

config = GraphRagConfig()
config.input_storage.base_dir = str(input_dir)
config.output_storage.base_dir = str(output)
config.cache.storage.base_dir = str(work_dir / "cache")
config.reporting.base_dir = str(work_dir / "logs")
config.update_output_storage.base_dir = str(work_dir / "update_output")
config.vector_store.db_uri = str(output / "lancedb")

config.completion_models["default_completion_model"] = ModelConfig(
    model_provider="openai",
    model="chat-standard",
    call_args={"extra_body": {"enable_thinking": False}},
    api_base="http://localhost:4000/v1",
    api_key="sk-zCjzfRyCk5KGoEzLvhK20Q",
)
config.embedding_models["default_embedding_model"] = ModelConfig(
    model_provider="openai",
    model="text-embedding",
    api_base="http://localhost:4000/v1",
    api_key="sk-zCjzfRyCk5KGoEzLvhK20Q",
)
dimensions = 1024  # 填写所用 Embedding 模型的实际输出维度
```

### 让索引用中文生成

GraphRAG 3.3.0 允许为不同的索引步骤分别指定提示词文件。本课用三个中文模板控制生成内容，模板中的占位符由 GraphRAG 在运行时填入。

| 索引步骤 | 模型要生成的内容 |
| --- | --- |
| [抽取实体和关系](assets/graph_rag/prompts/extract_graph_zh.txt) | 沿用原文中的中文片名、人名和角色名，用中文说明执导、参演、饰演等关系。保留解析器识别的 `"entity"` 和 `"relationship"` 标记。 |
| [汇总描述](assets/graph_rag/prompts/summarize_descriptions_zh.txt) | 同一人物或同一对实体有多条描述、需要模型合并时，用中文写出涉及哪些影片和分工。 |
| [生成社区报告](assets/graph_rag/prompts/community_report_zh.txt) | 标题、摘要、主要发现和评分理由使用中文；JSON 字段名和引用的记录编号保留原格式。 |

已有英文索引时，重新运行配置单元和下面的索引步骤，才能在输出表中看到新结果。

```python
prompts = work_dir / "prompts"
config.extract_graph.prompt = str(prompts / "extract_graph_zh.txt")
config.summarize_descriptions.prompt = str(prompts / "summarize_descriptions_zh.txt")
config.community_reports.graph_prompt = str(prompts / "community_report_zh.txt")
```

## 1. 加载原文

`resource/graph_rag/input/` 已有《奥本海默》《星际穿越》《盗梦空间》三份资料，每份介绍一部电影的导演、编剧、配乐和主要演员。先读《奥本海默》，找出哪句话同时提到了演员和他饰演的角色。

```python
print((input_dir / "奥本海默.txt").read_text(encoding="utf-8"))
```

`config.workflows` 指定本次执行的步骤，`api.build_index` 执行它。这里先把文本文件读成文档表。


```python
config.workflows = ["load_input_documents"]
_ = await api.build_index(config)

```

文档表中每行对应一部电影的原文。`title` 是文件名，`text` 是这部电影的完整资料。

```python
documents = pd.read_parquet(output / "documents.parquet")
documents

```

## 2. 切片

把文档切成文本单元：每片最多 500 个 token，相邻切片重叠 50 个 token。

`create_base_text_units` 切片；`create_final_documents` 把切片 ID 关联回文档。


```python
config.chunking.size = 500
config.chunking.overlap = 50
config.chunking.encoding_model = "o200k_base"

config.workflows = ["create_base_text_units", "create_final_documents"]
_ = await api.build_index(config)

```

查看切片正文、token 数和来源文档 ID。


```python
text_units = pd.read_parquet(output / "text_units.parquet")
text_units

```

## 3. 抽取实体和关系

《奥本海默》的原文写道：“希里安·墨菲饰演物理学家罗伯特·奥本海默”。这里有三个不同的实体：电影《奥本海默》、演员希里安·墨菲和角色罗伯特·奥本海默。可以记录演员参演了哪部电影，也可以记录他在这部电影里饰演了谁。

本课抽取三类实体：电影、人物（演职人员）和角色。导演、编剧、配乐和表演是人物在具体作品中承担的工作；同一个人可以承担多项工作。

`extract_graph` 抽取实体和关系并汇总描述，`finalize_graph` 整理图数据。

```python
config.extract_graph.entity_types = ["电影", "人物", "角色"]

config.workflows = ["extract_graph", "finalize_graph"]
_ = await api.build_index(config)

```

查看实体的名称、类型和描述。


```python
entities = pd.read_parquet(output / "entities.parquet")
entities

```

查看关系表。`source` 和 `target` 是关系两端的实体，`description` 用中文说明两者的联系。可以找找哪些关系提到了希里安·墨菲，以及每条关系对应的影片。

```python
relationships = pd.read_parquet(output / "relationships.parquet")
relationships

```

## 4. 划分社区

用图算法把联系紧密的实体分为一组。这一步不调用模型。


```python
config.workflows = ["create_communities"]
_ = await api.build_index(config)

```

查看社区编号、层级、父社区和成员数。`level=0` 是顶层，`parent=-1` 表示没有父社区。


```python
communities = pd.read_parquet(output / "communities.parquet")
communities[["community", "level", "parent", "size"]]

```

查看第一个社区里有哪些电影、演职人员和角色。多部影片可能共享导演或演员，社区的实际划分取决于抽取出的关系。

```python
member_ids = communities.iloc[0]["entity_ids"]
entities[entities["id"].isin(member_ids)][["title", "type"]]

```

关联回切片

把实体和关系的 ID 关联回切片，保留它们与原文之间的联系。


```python
config.workflows = ["create_final_text_units"]
_ = await api.build_index(config)

```

```python
text_units = pd.read_parquet(output / "text_units.parquet")
text_units[["id", "entity_ids", "relationship_ids"]].head()

```

## 5. 生成社区报告

Chat 模型读取上一步划分出的社区资料，使用配置好的中文报告模板，为社区起名、写摘要，并用具体的影片和人物关系说明主要发现。先看标题和摘要，再展开一份报告，对照前面的关系表核实它提到的合作。

```python
config.workflows = ["create_community_reports"]
_ = await api.build_index(config)

```

```python
reports = pd.read_parquet(output / "community_reports.parquet")
reports[["community", "title", "summary"]].head()

```

读一份完整报告。


```python
print(reports.iloc[0]["full_content"])

```

## 6. 向量化

Embedding 模型把文本转换为向量，供后续相似度搜索使用。


向量存入本地 LanceDB。各张表使用配置单元中填写的向量维度。`batch_size=10` 表示每批最多发送 10 条文本，可按服务限制调整。

```python
config.vector_store.vector_size = dimensions
for schema in config.vector_store.index_schema.values():
    schema.vector_size = dimensions
config.embed_text.batch_size = 10

```

对实体描述、社区报告和文本切片生成向量。


```python
config.workflows = ["generate_text_embeddings"]
_ = await api.build_index(config)

```

查看实体向量的条数、维度和前几个数值。


```python
import lancedb

db = lancedb.connect(config.vector_store.db_uri)
entity_vectors = db.open_table("entity_description").to_pandas()
print("实体向量条数：", len(entity_vectors))
print("每个向量的维度：", len(entity_vectors.iloc[0]["vector"]))
entity_vectors.iloc[0]["vector"][:8]

```

完成后，可在 `resource/graph_rag/output/` 查看各阶段生成的 Parquet 表，在其 `lancedb/` 子目录查看向量库文件。

