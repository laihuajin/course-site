# Graph RAG - 检索与生成

> 《奥本海默》中饰演奥本海默的演员，在资料里的另一部电影中饰演了谁？

## 准备阶段

```python
from pathlib import Path

import graphrag.api as api
import pandas as pd
from graphrag.config.models.graph_rag_config import GraphRagConfig
from graphrag_llm.config import ModelConfig
from IPython.display import Markdown, display

project_dir = Path.cwd()
if project_dir.name == "课件":
    project_dir = project_dir.parent
work_dir = project_dir / "resource" / "graph_rag"
output = work_dir / "output"

text_units = pd.read_parquet(output / "text_units.parquet")
entities = pd.read_parquet(output / "entities.parquet")
relationships = pd.read_parquet(output / "relationships.parquet")
communities = pd.read_parquet(output / "communities.parquet")
reports = pd.read_parquet(output / "community_reports.parquet")

print(f"原文切片 {len(text_units)} 段")
```

```python
config = GraphRagConfig()
config.vector_store.db_uri = str(output / "lancedb")
config.reporting.base_dir = str(work_dir / "logs")
dimensions = 1024
config.vector_store.vector_size = dimensions
for schema in config.vector_store.index_schema.values():
    schema.vector_size = dimensions

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
```

## 1. Basic Search · 直接找原文

先用熟悉的向量检索作对照。Basic 把**问题和原文切片**按向量相似度匹配，把选中的切片交给模型写答案。题目只提到《奥本海默》，要答出另一部影片里的角色，模型需要同时看到哪两段资料？

这次取两段切片。运行后先读回答，再看它拿到了哪两段原文。

```python
casting_question = "《奥本海默》中饰演奥本海默的演员，在资料里的另一部电影中饰演了谁？请写出两部片名、演员和对应角色。"
answer_style = "用中文简短回答，保留证据编号。材料不足时指出缺少哪一段证据。"
config.basic_search.k = 2

basic_answer, basic_context = await api.basic_search(
    config=config,
    text_units=text_units,
    query=casting_question,
    response_type=answer_style,
)
display(Markdown(basic_answer))
```

```python
print(basic_context["sources"].to_string(index=False))
```

核对答案时，一段原文应当能确认希里安·墨菲饰演了奥本海默，另一段应当能确认他在《盗梦空间》里饰演费舍尔。如果两段都找到了，Basic 已经能答对；如果少了一段，即使回答里写出了正确片名，也要区分哪一半有依据。

## 2. Local Search · 围绕实体找证据

保持问题不变，把向量匹配的对象换成**实体描述**。例如，《奥本海默》这部电影和罗伯特·奥本海默这个角色都可能成为入口。Local 用匹配到的实体收集相关关系、原文切片和社区报告，在上下文预算内选取材料，再让模型作答。这里最值得找的是能把两份原文接起来的演员。

```python
local_answer, local_context = await api.local_search(
    config=config,
    entities=entities,
    relationships=relationships,
    text_units=text_units,
    communities=communities,
    community_reports=reports,
    community_level=0,
    covariates=None,  # 第 44 节没有单独抽取声明
    query=casting_question,
    response_type=answer_style,
)
display(Markdown(local_answer))
```

先找这次上下文里与希里安·墨菲直接相连的关系，再读它附带的原文。关系的两端可能是演员和影片，也可能是演员和角色，具体以这次索引为准。

```python
edges = local_context["relationships"]
actor = "希里安·墨菲"
actor_edges = edges["source"].str.contains(actor) | edges["target"].str.contains(actor)
display(edges.loc[actor_edges, ["id", "source", "target", "description"]])

for row in local_context["sources"].itertuples():
    print(f"[{row.id}] {row.text}\n")
```

把两个答案并排想一想。如果都答对了，Local 多给出了哪些可以核对的关系？如果上表为空，先在切片里找演员姓名，再看实体表里用了什么写法。关系未进入这次上下文，仍然可以回到原文核实。

## 3. Global Search · 从社区报告归纳片库

> 三部电影在导演、编剧和配乐上有哪些稳定的合作，又有哪些变化？

这次没有指定从哪位创作者查起。Global 可以从上一节生成的**社区报告**入手。这里固定读取顶层报告，按上下文容量分批。**Map** 阶段让模型从每批报告中提取与问题有关的要点并评分；**Reduce** 阶段汇集得分较高、能放进上下文的要点，组织成最终回答。报告少时，一批就可能装下。

```python
global_answer, global_context = await api.global_search(
    config=config,
    entities=entities,
    communities=communities,
    community_reports=reports,
    community_level=0,
    dynamic_community_selection=False,
    query="比较资料中三部电影的导演、编剧与配乐分工。哪些人反复合作，哪里换了人？用片名举例。",
    response_type="用中文按导演、编剧、配乐分三点回答，注明报告编号。资料未说明的地方请指出。",
)
display(Markdown(global_answer))
```

查看本轮的报告标题，再选“配乐”这一点对照原文。原文中《盗梦空间》和《星际穿越》都写了汉斯·季默，《奥本海默》写的是路德维希·戈兰松。如果回答少举了一部片，读对应的报告，看看这个细节在生成报告时是否就已经被省略了。

```python
used_reports = global_context["reports"]
display(used_reports[["id", "title"]])
print(used_reports.iloc[0]["content"])  # 改行号可读另一份报告
```
