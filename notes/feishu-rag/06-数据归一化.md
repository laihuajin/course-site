```python
from deal_pilot.feishu.api import WikiAPI

wiki = WikiAPI()

await wiki.list_spaces()
```

```python
from deal_pilot.feishu.api import WikiAPI

wiki = WikiAPI()

await wiki.get_space("7696739752510246187")
```

```python
from deal_pilot.feishu.api import WikiAPI

wiki = WikiAPI()
from rich import print
nodes = await wiki.list_nodes("7696739752510246187", parent_node_token="QPmAwpxKoi46Q9kMKw6cQbavnod")
print(nodes)
```

```python
from deal_pilot.feishu.api import WikiAPI

wiki = WikiAPI()

async for node in wiki.list_all_space_nodes():
    print(node)
```

```python
from deal_pilot.feishu.api import DocxAPI

docx = DocxAPI()

doc = await docx.get_document("DVEDdIdqtoGCHvxIA1bcrXNJnwh")
print(doc)
```

```python
from deal_pilot.feishu.api import DocxAPI

docx = DocxAPI()

doc = await docx.list_blocks('DVEDdIdqtoGCHvxIA1bcrXNJnwh')
print(doc)
```
