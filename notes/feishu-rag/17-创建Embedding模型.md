```python
from deal_pilot.rag.embedding_model import create_embedding_model

model = create_embedding_model()
```

```python
vectors = await model.aembed_documents(["asdfasddfadsfdasdfasdfsadf", "士大夫撒单位放入阿斯顿发艾师傅艾师傅的是"])
```

```python
len(vectors)
```

```python
[
    len(v)
    for v in  vectors
]
```

```python
vectors[0]
```
