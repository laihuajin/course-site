# Thread API

可以通过 `Agent Server` 暴露的 `Thread API` 管理线程（图的持久化状态）

API文档地址：http://localhost:2024/docs#tag/threads

API调用：推荐 `langgraph_sdk`

本节先补充上节课遗漏的"子图"接口，再讲解 Thread 接口。

## 安装 LangGraph SDK

上一节课已经安装过 `langgraph-sdk`，这里重复执行也无副作用，仅保证课件可独立运行：

```python
!uv add langgraph-sdk==0.4.2
```

## 创建客户端

连接本地 Agent Server（默认端口 2024），获得 `client.threads` 子客户端：

```python
from langgraph_sdk import get_client

# 连接本地 Agent Server
client = get_client(url="http://localhost:2024")
client
```

## 补充：获取子图

> GET /assistants/{assistant_id}/subgraphs
>
> GET /assistants/{assistant_id}/subgraphs/{namespace}

子图接口返回的是图中**结构化定义**的子图节点。当前注册的 `agent` 图没有结构化子图（其子 Agent 是通过工具动态调用编译的子图，不算结构化子图），因此返回空对象。

先获取一个已有 Assistant 的 ID：

```python
# 获取一个已有的 agent Assistant
assistants = await client.assistants.search(graph_id="agent", limit=1)
assistant_id = assistants[0]["assistant_id"]
assistant_id
```

```python
# 当前图没有结构化子图，返回空对象
subgraphs = await client.assistants.get_subgraphs(assistant_id)
subgraphs
```

```python
# 按命名空间查询子图，同样为空
subgraphs = await client.assistants.get_subgraphs(assistant_id, namespace="child:graph")
subgraphs
```

## 创建线程

> POST /threads

创建线程时可附带元数据，并通过 `graph_id` 关联图：

```python
# 创建线程
# metadata：附加元数据，可用于后续搜索过滤
thread = await client.threads.create(
    metadata={
        "title": "新会话",
        "user_id": "007",
        "graph_id": "agent",
    },
)
thread
```

极少用的字段说明（需要时查询）：

- ttl（Time-To-Live）: 可以控制线程存活周期，比如1个月后自动清理线程数据
- supersteps：在初始化线程时，可以直接传递每一个检查点的状态

保存 Thread ID 供后续接口使用：

```python
thread_id = thread["thread_id"]
thread_id
```

## 获取线程

> GET /threads/{thread_id}

通过 ID 获取线程的详细信息：

```python
thread = await client.threads.get(thread_id)
thread
```

返回字段说明：

- state_updated_at：线程状态最近更新时间（每次运行产生新检查点时变化）
- status：线程状态，如 idle（空闲）/ busy（执行中）/ interrupted（中断）/ error（出错）
- config：最新运行使用的配置，来自 assistant 的 config
- values：线程当前的图状态值（如消息列表），未运行过时为 None

## 更新线程

> PATCH /threads/{thread_id}

更新线程的元数据（`metadata`）等字段，新的元数据会与已有元数据合并：

```python
thread_v2 = await client.threads.update(
    thread_id,
    metadata={"title": "langgraph_sdk的使用方法"},
)
thread_v2
```

返回字段与「获取线程」一致，区别在于其 `metadata` 已与传入的新元数据合并（旧的 `title`、`graph_id` 等会保留）。

## 搜索线程

> POST /threads/search

按 `metadata`、`values`、`status` 等过滤，支持分页与排序。该接口同样用于列出全部线程：

```python
# 按 metadata 过滤
result = await client.threads.search(
    metadata={"user_id": "007"},
    limit=10,
)
result
```

```python
# 使用 select 指定返回字段，并按创建时间倒序
result = await client.threads.search(
    limit=5,
    offset=0,
    sort_by="created_at",
    sort_order="desc",
    select=["thread_id", "status", "created_at"],
)
result
```

返回的是线程对象列表，每个元素的字段与「获取线程」返回的结构一致；若指定了 `select`，则只返回选中的字段。

## 统计线程数量

> POST /threads/count

按条件统计线程数量：

```python
count = await client.threads.count()
print(f"线程数量: {count}")
```

## 复制线程

> POST /threads/{thread_id}/copy

复制线程会**连同状态和检查点一起复制**，得到一个新的线程。

下面从服务器上已有的、有运行记录的线程复制一份，得到的线程自带状态数据，正好用于接下来的"状态与历史"演示：

```python
source_id = "019fef6f-fc1f-72b2-a824-98a9c8fc4cda"
# 复制线程（连同状态与检查点一起复制）
copy_thread = await client.threads.copy(source_id)
copy_thread_id = copy_thread["thread_id"] # type: ignore
copy_thread_id
```

返回一个新的线程对象，字段与「获取线程」一致，`thread_id` 为新生成（旧线程不受影响）。

## 获取线程状态

> GET /threads/{thread_id}/state

获取线程的最新状态（即最新检查点的状态）。这里用上面复制出的线程演示（它自带状态数据）：

```python
state = await client.threads.get_state(copy_thread_id)
import json
print(json.dumps(state, indent=2, ensure_ascii=False))
```

返回字段说明：

- values：图的最新状态（如消息列表 messages）
- next：下一步要执行的节点列表；空列表表示图已执行完毕
- tasks：正在排队/执行中的任务
- metadata：本次运行的元数据（system_prompt、graph_id、assistant_id 等）
- checkpoint：最新检查点信息（checkpoint_id、thread_id、checkpoint_ns）
- checkpoint_id：最新检查点 ID，每个超步产生一个
- parent_checkpoint / parent_checkpoint_id：上一个检查点
- interrupts：当前的中断（human-in-the-loop 触发 interrupt 时才会有）
- created_at：检查点创建时间

## 获取线程历史

> GET /threads/{thread_id}/history

获取线程的全部历史状态（每个超步产生一个检查点，对应一个状态）：

```python
history = await client.threads.get_history(copy_thread_id, limit=10)
checkpoints = [state["checkpoint"] for state in history]
print(json.dumps(history, indent=2, ensure_ascii=False))
```

返回的是状态历史列表，每个元素对应一次超步执行后产生的一个检查点，字段与「获取线程状态」返回的结构一致，其中 `metadata.step` 为超步序号，`checkpoint` 里是 `checkpoint_id` 等检查点信息。

## 获取指定检查点状态

> GET /threads/{thread_id}/state/{checkpoint_id}

通过 `checkpoint_id` 获取某个历史检查点的状态。先取历史中的一个检查点 ID：

```python
# 获取指定检查点的状态
state = await client.threads.get_state(copy_thread_id, checkpoint_id="1f1954b1-a620-642e-8000-8ca661c7fcb5")
state
```

返回字段与「获取线程状态」一致，只是对应的是指定检查点那一刻的状态（可用于回溯/时间旅行）。

## 清理线程

> POST /threads/prune

按 ID 清理线程。`strategy="delete"` 会删除整个线程，`strategy="keep_latest"` 则只清理旧检查点但保留线程及最新状态。

先创建一个临时线程用于清理演示：

```python
# 清理临时线程（delete 策略会删除整个线程）
result = await client.threads.prune(
    thread_ids=[copy_thread_id],
    strategy="keep_latest",
)
result
```

返回字段说明：

- pruned_count：实际清理的线程数量

## 删除线程

> DELETE /threads/{thread_id}

删除线程，其所有检查点也会一并删除。删除后再次查询会抛出 `NotFoundError`：

```python
from langgraph_sdk.errors import NotFoundError

await client.threads.delete(copy_thread_id)

try:
    await client.threads.get(copy_thread_id)
except NotFoundError as e:
    print("已删除：", e)
```
