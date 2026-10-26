# LangChain 文本切片工具

将长文本拆成多个 chunk，便于后续向量化和检索。本节学习两个通用文本切分器：

- `CharacterTextSplitter`：按一个指定分隔规则拆分，再合并短片段。
- `RecursiveCharacterTextSplitter`：按分隔符优先级拆分，过长的局部继续细分。

按顺序运行单元格：准备文本 → 普通切分 → 递归切分 → 重叠实验 → 切分 Documents。

全部示例使用本地自编文本，无需模型、API Key 或外部服务。选择项目的 Python 内核；项目已有 `langchain-text-splitters`，本课无需安装依赖。

```python
from importlib.metadata import version
from textwrap import fill

from langchain_core.documents import Document
from langchain_text_splitters import CharacterTextSplitter, RecursiveCharacterTextSplitter

print("langchain-text-splitters:", version("langchain-text-splitters"))
```

## 准备一篇长文本

下面是一篇虚构的企业知识库使用指南。段落之间用空行 `\n\n` 分隔，既有短段落，也有明显较长的段落，方便观察两种切分器的区别。

```python
text = """企业知识库使用指南
销售团队每天需要查询产品资料、报价规则和客户案例。过去，这些内容散落在聊天记录和个人文件夹中，新同事往往不知道该从哪里开始寻找。建设统一知识库的第一步，是把这些资料整理到可以持续维护的文档中。

知识库中的每篇文档都应注明负责人、适用范围和更新时间。负责人负责核对内容是否准确，适用范围帮助读者判断规则是否与当前业务有关，更新时间则提醒读者注意信息是否已经过期。已经失效的规则要及时归档，避免与新规则混在一起。

新员工第一次使用知识库时，可以先阅读入门导航，了解产品、销售流程和常见问题分别存放在哪里。遇到客户询问产品功能时，应先确认客户使用的版本，再查找对应版本的功能说明，不要直接把其他版本的能力当作当前承诺。遇到价格问题时，应同时阅读基础报价、折扣条件和审批要求，因为一条单独的价格记录通常不能说明最终成交条件。准备客户方案时，可以参考历史案例，但需要检查案例中的行业、规模和部署方式是否与当前客户相近，不能直接复制其中的结论。如果搜索没有找到答案，应记录已经使用的关键词和仍然缺失的信息，再向文档负责人反馈，这样可以帮助维护者补充真正有价值的内容。负责人补充文档后，还应把新内容加入导航，并说明它解决了什么问题，让后续读者能够沿着同一条路径找到答案。

客户案例应保留问题背景、实施过程和最终结果。只有结果而没有前提的案例容易被误用，因此维护者需要说明方案成立的条件，也要记录不能直接推广到其他客户的限制。涉及客户隐私的内容应按公司的实际制度处理。

为知识库增加智能问答时，需要先把文档读取成文本，再切成大小合适的片段。片段过长，检索结果可能包含大量无关内容；片段过短，问题的条件与答案可能被分到不同位置。切片时应同时考虑长度和自然语言边界。

切片完成后，可以为每个片段生成向量，并保存文档来源等元数据。收到问题时，系统检索相关片段，再把片段作为上下文交给模型。回答中保留来源信息，可以帮助使用者回到原文核对规则，也方便维护者定位需要更新的资料。

上线前应准备一组来自真实业务的问题，逐条检查检索结果是否包含回答所需的信息。除了直接询问产品功能的问题，还应加入需要同时理解限制条件、操作步骤和例外情况的问题。发现回答遗漏条件时，要先查看原文是否已经写清楚，再检查切片边界是否把条件和结论分开，最后检查检索是否找到了相关片段。调整切片长度或重叠量后，应使用同一组问题重新比较结果，不能只凭某一个回答看起来更顺畅就认定方案更好。知识库内容发生更新时，也需要同步更新相关片段和索引，并检查旧规则是否仍然出现在回答中。持续记录这些检查结果，才能知道一次调整究竟改善了哪些问题，又带来了哪些新的限制。"""

paragraphs = text.split("\n\n")
print("全文字符数:", len(text))
print("段落数:", len(paragraphs))
print("各段字符数:", [len(p) for p in paragraphs])
```

## 先认识长度和重叠参数

| 参数 | 含义 |
| --- | --- |
| `chunk_size` | 目标切片长度上限；普通切分器遇到过长的基本片段时可能超限 |
| `chunk_overlap` | 相邻切片的目标重叠长度；实际受片段边界约束 |
| `length_function` | 长度计算函数；默认 `len`，按字符计数 |

本课先用 `chunk_size=180`、`chunk_overlap=0`，看清切分边界后再加入重叠。
这里的长度是 Python `len()`，不是字节数，也不是模型 token 数。示例参数用于观察行为，不代表业务最优值。

```python
chunk_size = 180


def show_chunks(chunks: list[str]) -> None:
    """打印切片内容；换行排版只影响显示，不修改原切片。"""
    print(f"共 {len(chunks)} 个切片，长度: {[len(chunk) for chunk in chunks]}")
    for index, chunk in enumerate(chunks, start=1):
        print(f"\n--- 切片 {index} | {len(chunk)} 字符 ---")
        print(fill(chunk, width=60))
```

## 普通文本切分器：CharacterTextSplitter

`separator` 是一个分隔规则，默认是 `"\n\n"`。本例按空行拆成段落，再按顺序合并能放进同一块的短段落；合并时分隔符也计入长度。

这里显式设置 `keep_separator="end"`，把分隔符保留在前一个片段末尾，方便观察；普通切分器默认不保留分隔符，而会在合并时重新插入。默认 `strip_whitespace=True` 会去掉最终切片两端的空白。

```python
character_splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=chunk_size,
    chunk_overlap=0,
    length_function=len,
    keep_separator="end",
)

character_chunks = character_splitter.split_text(text)
show_chunks(character_chunks)
```

### 改变 separator，改变基本切分单位

把分隔符改为中文句号 `。`，同一段文字就可以按句子拆分，再合并成片。
`keep_separator="end"` 将句号保留在前一句末尾；但超过长度的单个长句仍可能超限。

`separator` 默认按普通字符串处理；只有设置 `is_separator_regex=True`，才会把它当作正则表达式。

```python
sentence_splitter = CharacterTextSplitter(
    separator="。",
    chunk_size=chunk_size,
    chunk_overlap=0,
    keep_separator="end",
)

sentence_chunks = sentence_splitter.split_text(text)
show_chunks(sentence_chunks)
```

### 如果就是要按固定字符数切分

设置 `separator=""`，每个字符都会成为基本单元，再合并成片。无重叠时，除最后一片外通常都会填满 `chunk_size`，但可能从句子中间断开。

```python
fixed_splitter = CharacterTextSplitter(
    separator="",
    chunk_size=chunk_size,
    chunk_overlap=0,
)

fixed_chunks = fixed_splitter.split_text(text)
show_chunks(fixed_chunks)
```

## 递归文本切分器：RecursiveCharacterTextSplitter

递归切分器使用 `separators` 列表，默认优先级为：

```text
["\n\n", "\n", " ", ""]
     空行 → 换行 → 空格 → 字符
```

先选择当前文本中存在的优先分隔符。短片段按顺序合并，过长的局部用后续更细的分隔符继续拆分。它不会把全文不加区别地依次切四遍。

默认规则包含末尾的 `""`，在本课按字符计数的设置下，可以把过长文本继续拆到长度范围内。

```python
recursive_splitter = RecursiveCharacterTextSplitter(
    chunk_size=chunk_size,
    chunk_overlap=0,
    length_function=len,
    keep_separator="end",
)

recursive_chunks = recursive_splitter.split_text(text)
show_chunks(recursive_chunks)
```

### 默认规则对中文有什么影响？

默认列表没有中文句号。刚才的长段落没有换行和空格，所以最终会降到字符级，可能在一句话中间切开。

可以加入中文标点，让过长段落先尝试按句子、短句切，再退到字符。

```python
chinese_separators = ["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""]

chinese_splitter = RecursiveCharacterTextSplitter(
    separators=chinese_separators,
    chunk_size=chunk_size,
    chunk_overlap=0,
    keep_separator="end",
)

default_paragraph_chunks = recursive_splitter.split_text(text)
chinese_paragraph_chunks = chinese_splitter.split_text(text)

print("默认规则：")
for index, chunk in enumerate(default_paragraph_chunks, start=1):
    print(f"切片 {index} | {len(chunk)} 字符 | 结尾: {chunk[-25:]!r}")

print("\n加入中文标点：")
show_chunks(chinese_paragraph_chunks)
```

### 分隔符列表的顺序代表优先级

`["。", "！", "？"]` 表示先尝试句号；它不是“同时识别三种句末符号”。如果希望这些标点具有同一级别，可以把它们放在一个正则规则中：

```python
regex_splitter = RecursiveCharacterTextSplitter(
    separators=[r"\n\n", r"\n", r"[。！？]", r"[；，]", r"\s", ""],
    is_separator_regex=True,
    keep_separator="end",
    chunk_size=12,
    chunk_overlap=0,
)

mixed_text = "先确认客户使用的版本！再检查功能限制？最后核对报价条件。"
print(regex_splitter.split_text(mixed_text))
```

### 保留字符级兜底

如果自定义列表中去掉 `""`，遇到没有匹配分隔符的长文本，递归切分器也可能返回超长切片。长度约束能否满足，取决于分隔规则是否允许继续拆分。

```python
no_fallback_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n"],
    chunk_size=chunk_size,
    chunk_overlap=0,
)

print("没有字符兜底:", [len(c) for c in no_fallback_splitter.split_text(text)])
print("保留字符兜底:", [len(c) for c in recursive_splitter.split_text(text)])
```

## 加入 chunk_overlap，保留一点相邻上下文

先用短字符串看清机制。`chunk_size=10`、`chunk_overlap=3` 时，字符级合并可以让上一片末尾的 3 个字符出现在下一片开头。

两个类都支持这个参数。设置时 `chunk_size` 必须大于 0，`chunk_overlap` 不能为负或大于 `chunk_size`；通常选 `0 <= chunk_overlap < chunk_size`，让每片有空间放新内容。

```python
sequence = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ"

demo_splitter = RecursiveCharacterTextSplitter(
    separators=[""],
    chunk_size=10,
    chunk_overlap=3,
)
demo_splitter.split_text(sequence)
```

### 实际重叠可能更少，甚至为零

合并时保留的是完整的基本片段。如果一个句子已经长于目标重叠量，切分器通常不会为了凑重叠而额外截取半句。

下面每句话 9 个字符。每片可以放两句话，但目标重叠只有 3 个字符，放不下一句，因此实际没有重叠。

```python
overlap_text = "甲组负责整理资料。乙组负责核对版本。丙组负责审批价格。丁组负责维护案例。"
boundary_splitter = RecursiveCharacterTextSplitter(
    separators=["。", ""],
    keep_separator="end",
    chunk_size=18,
    chunk_overlap=9,
)
boundary_splitter.split_text(overlap_text)
```
