# Document Loader

将不同来源的数据加载为统一的 `Document`：`page_content` 保存正文，`metadata` 保存来源等信息。

## 准备

先选择项目的 Python 内核，运行下面的单元格安装依赖。安装后如遇到导入问题，重启内核再继续。

PDF 使用项目已有的 `pypdf`。示例文件位于 `resource/document_loader/`，PDF 复用 `resource/ECMA.pdf`。

```python
import warnings

warnings.filterwarnings(
    "ignore",
    message=r"`langchain-community` is being sunset.*",
    category=DeprecationWarning,
)
```

```python
import sys

sys.executable
```

```python
import sys

# 安装到当前 Notebook 内核使用的 Python 环境
!uv pip install --python "{sys.executable}" langchain-community "unstructured[md]" jq
```

```python
from pathlib import Path

# 兼容从项目根目录或课件目录启动 Notebook
project_dir = Path.cwd()
if not (project_dir / "pyproject.toml").exists():
    project_dir = project_dir.parent

resource_dir = project_dir / "resource"
sample_dir = resource_dir / "document_loader"
```

## 纯文本：TextLoader

整个文本文件加载为一个 `Document`，指定 UTF-8 编码。

```python
from langchain_community.document_loaders import TextLoader

text_loader = TextLoader(sample_dir / "sales_policy.txt", encoding="utf-8")
text_docs = text_loader.load()

len(text_docs)
```

```python
print(text_docs[0].page_content)
text_docs[0].metadata
```

## Markdown：UnstructuredMarkdownLoader

`mode="elements"` 按标题、段落、列表项等元素输出多个 `Document`。正文是解析后的文本，元素类别记录在 `metadata["category"]` 中，不保留原始 Markdown 标记。

首次运行时，Unstructured 会自动下载其 spaCy 文本分析模型，需要联网。

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

markdown_loader = UnstructuredMarkdownLoader(
    str(sample_dir / "sales_faq.md"),
    mode="elements",
)
markdown_docs = markdown_loader.load()

len(markdown_docs)
```

```python
for doc in markdown_docs:
    print(doc.metadata.get("category"), "|", doc.page_content)

markdown_docs[0].metadata
```

## CSV：CSVLoader

每条数据行生成一个 `Document`，列名和值组成正文；`metadata["row"]` 从 0 开始。

```python
from langchain_community.document_loaders import CSVLoader

csv_loader = CSVLoader(
    file_path=str(sample_dir / "products.csv"),
    encoding="utf-8",
)
csv_docs = csv_loader.load()

len(csv_docs)
```

```python
print(csv_docs[0].page_content)
csv_docs[0].metadata
```

## JSON：JSONLoader

`jq_schema=".faqs[].answer"` 提取每条问答的 `answer`，每个答案生成一个 `Document`；默认元数据包含文件来源和从 1 开始的 `seq_num`。

```python
from langchain_community.document_loaders import JSONLoader

json_loader = JSONLoader(
    file_path=sample_dir / "faq.json",
    jq_schema=".faqs[].answer",
    text_content=True,
)
json_docs = json_loader.load()

len(json_docs)
```

```python
print(json_docs[0].page_content)
json_docs[0].metadata
```

## PDF：PyPDFLoader

`mode="page"` 每页生成一个 `Document`，`metadata["page"]` 从 0 开始。

示例通过 `load()` 加载全部页面。这里提取 PDF 文本层；扫描件的识字需要另行配置 OCR。

```python
from langchain_community.document_loaders import PyPDFLoader

pdf_loader = PyPDFLoader(str(resource_dir / "ECMA.pdf"), mode="page")
pdf_docs = pdf_loader.load()

len(pdf_docs)
```

```python
print(pdf_docs[100].page_content[:800])
pdf_docs[100].metadata
```

## 参考

- [LangChain Document Loaders](https://docs.langchain.com/oss/python/integrations/document_loaders/)
- [Community Loader API](https://reference.langchain.com/python/langchain-community/document_loaders)
