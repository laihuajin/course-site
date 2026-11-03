```python
from typing import Literal,get_args


type MilvusCollection = Literal["feishu_doc", "memory", "feishu_excel", "feishu_chat"]
{ name: 123  for name in get_args(MilvusCollection.__value__)}
```

```python
from deal_pilot.rag.store.feishu_doc.init_store import init_store

await init_store()
```
