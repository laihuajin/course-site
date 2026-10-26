# Run API

可以通过 `Agent Server` 暴露的 `Run API` 管理运行（图的一次调用）

API文档地址：http://localhost:2024/docs#tag/thread-runs

API调用：推荐 `langgraph_sdk`

运行分两类：
- **有状态运行（Thread Runs）**：在某个线程上执行，会更新线程的状态与检查点
- **无状态运行（Stateless Runs）**：不绑定线程，运行结束即清理，无记忆

> 流式接口（`/runs/stream`）依赖"流式处理"概念，本节不讲解。

## 安装 LangGraph SDK

上一节课已经安装过 `langgraph-sdk`，这里重复执行也无副作用，仅保证课件可独立运行：

```python
!uv add langgraph-sdk==0.4.2
```

## 创建客户端

连接本地 Agent Server（默认端口 2024），获得 `client.runs` 子客户端：

```python
from langgraph_sdk import get_client

# 连接本地 Agent Server
client = get_client(url="http://localhost:2024")
client
```

## 准备工作

有状态运行需要绑定一个线程。先获取一个已有的 `agent` Assistant，再创建一个线程：

```python
assistant_id = "efbf07f8-c28f-4db8-a7ff-17b58c4af012"

# 创建一个线程，用于有状态运行
thread = await client.threads.create(
    metadata={
        "__name__": "第20节课：Run API测试"  # langsmith studio 会自动读取该值
    }
)
thread_id = thread["thread_id"]
```

## 创建运行

### 创建后台运行

> POST /threads/{thread_id}/runs

创建运行后**立即返回**（后台运行），不等待执行完成。适合"先创建、稍后取结果"或无人值守的场景：

```python
# 创建后台运行，立即返回，不等待执行完成
run = await client.runs.create(
    thread_id=thread_id,
    assistant_id=assistant_id,
    input={"messages": [{"role": "user", "content": "你好"}]},
    metadata={"lesson": "第20节课的运行测试"},
)
run_id = run["run_id"]
run
```

### 等待运行完成

> GET /threads/{thread_id}/runs/{run_id}/join

阻塞等待某个**已创建**的后台运行执行完成，运行完后返回图的最新状态

```python
# 阻塞等待运行执行完成，返回最终状态（values）
final_state = await client.runs.join(thread_id, run_id)
final_state
```

### 创建并等待运行完成

> POST /threads/{thread_id}/runs/wait

一步完成"创建运行 + 等待执行完成"，直接返回最终状态。适合需要同步拿到结果的场景：

```python
# 创建运行并等待执行完成，一步到位
result = await client.runs.wait(
    thread_id=thread_id,
    assistant_id=assistant_id,
    input={"messages": [{"role": "user", "content": "第二个问题：地球为什么是圆的？"}]}
)
result
```

`wait` 与 `create + join` 的效果等价，只是封装成了一个接口。

### 创建运行的可配置字段

- `thread_id`: 指定运行的线程
- `assistant_id`: 指定运行的AI助理
- `input`: 注入的状态
- `metadata`：本次运行的元数据，可用于后续搜索、过滤
- `config` / `context`：指定这次运行使用的配置，会覆盖 `assistant` 的对应配置
- `after_seconds`：指定多少秒后开始运行，用于定时运行
- `if_not_exists`：线程不存在时的处理，`reject`（默认，报错）/ `create`（自动创建线程）
- `durability`：控制检查点（checkpoint）写入的频率

  | 模式            | 行为                                                 | 代价                                             |
  | :-------------- | :--------------------------------------------------- | :----------------------------------------------- |
  | `exit`          | 只在图**退出时**（成功 / 出错 / 中断）持久化一次状态 | 性能最好，但中途崩溃则中间状态全丢，无法恢复     |
  | `async`（默认） | 下一步执行的同时**异步**后台写入检查点               | 兼顾性能与可靠，但崩溃时最后一个检查点可能没写入 |
  | `sync`          | 下一步开始前**同步**写入检查点                       | 最可靠（每步都落盘），性能开销最大               |


### run 对象字段说明

- run_id：运行 ID
- thread_id：运行所属的线程
- assistant_id：本次运行使用的助手
- status：运行状态，刚创建时为 `pending`（排队中）
- metadata：本次运行的元数据
- kwargs：创建运行时的参数（input、config、context 等）

## 运行管理

### 获取运行

> GET /threads/{thread_id}/runs/{run_id}

通过 run_id 获取运行的最新信息，最常用的是查看 `status`（运行是否完成）：

```python
# 通过 run_id 获取运行的最新信息
run = await client.runs.get(thread_id, run_id)
print(f"运行状态: {run['status']}")
run
```

### 列出运行

> GET /threads/{thread_id}/runs

列出线程上的运行记录，支持分页、按状态过滤、指定返回字段。该接口同时用于列出线程上的全部运行：

```python
# 列出该线程上的运行记录
runs = await client.runs.list(thread_id, limit=10)
import json
print(json.dumps(runs, indent=2, ensure_ascii=False))
```

```python
# 按状态过滤 + 指定返回字段
runs = await client.runs.list(
    thread_id,
    limit=10,
    status="success",
    select=["run_id", "status", "created_at"],
)
runs
```

### 删除运行

> DELETE /threads/{thread_id}/runs/{run_id}

删除运行记录，删除后再次查询会抛出 `NotFoundError`：

```python
from langgraph_sdk.errors import NotFoundError

# 删除运行
await client.runs.delete(thread_id, run_id)

try:
    await client.runs.get(thread_id, run_id)
except NotFoundError as e:
    print("已删除：", e)
```

```python
# 删除后，检查点也会被一并删除
await client.threads.get_state(thread_id=thread_id)
```

## 无状态运行（Stateless Runs）

无状态运行**不绑定已有线程**。服务器会为每次运行临时创建线程，运行结束后默认删除（`on_completion="delete"`）。

特点：
- 无记忆：每次运行相互独立，不共享状态
- 轻量：无需先创建线程，适合一次性调用

### 创建无状态后台运行

> POST /runs

与有状态后台运行的区别：不传 `thread_id`：

```python
# 创建无状态后台运行（不传 thread_id）
run = await client.runs.create(
    thread_id=None,
    assistant_id=assistant_id,
    input={"messages": [{"role": "user", "content": "无状态运行：一次性的问答"}]},
)
print(f"状态: {run['status']}")
run
```

### 无状态等待输出

> POST /runs/wait

创建无状态运行并直接等待输出，常用于在代码里同步调用一次图：

```python
# 创建无状态运行并等待输出
result = await client.runs.wait(
    thread_id=None,
    assistant_id=assistant_id,
    input={"messages": [{"role": "user", "content": "无状态等待输出"}]},
)
result
```

### 批量创建运行

> POST /runs/batch

一次创建多个无状态后台运行，立即返回运行列表：

```python
# 一次创建多个无状态后台运行，立即返回
runs = await client.runs.create_batch([
    {"assistant_id": assistant_id, "input": {"messages": [{"role": "user", "content": "批次 1"}]}},
    {"assistant_id": assistant_id, "input": {"messages": [{"role": "user", "content": "批次 2"}]}},
    {"assistant_id": assistant_id, "input": {"messages": [{"role": "user", "content": "批次 3"}]}},
]) # type: ignore
print(f"创建数量: {len(runs)}")
print([(r["run_id"], r["status"]) for r in runs])
```
