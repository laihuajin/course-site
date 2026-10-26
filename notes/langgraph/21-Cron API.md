# Cron API

可以通过 `Agent Server` 暴露的 `Cron API` 按指定的时间计划**定时运行**图（类似操作系统的 `cron` 定时任务）

API文档地址：http://localhost:2024/docs#tag/crons

API调用：推荐 `langgraph_sdk`

Cron 分两类：
- **无状态 Cron（Stateless Crons）**：每次执行都新建一个线程，运行结束后删除
- **有状态 Cron（Thread Crons）**：绑定一个**固定的线程**，每次执行都复用该线程，可以积累上下文

## 安装 LangGraph SDK

上一节课已经安装过 `langgraph-sdk`，这里重复执行也无副作用，仅保证课件可独立运行：

```python
!uv add langgraph-sdk==0.4.2
```

## 创建客户端

连接本地 Agent Server（默认端口 2024），获得 `client.crons` 子客户端：

```python
from langgraph_sdk import get_client

# 连接本地 Agent Server
client = get_client(url="http://localhost:2024")
client
```

## 准备工作

需要准备一个 `agent` Assistant（用来定时执行的图），以及一个线程（用于有状态 Cron 的演示）：

```python
assistant_id = "efbf07f8-c28f-4db8-a7ff-17b58c4af012"

# 创建一个线程，用于有状态 Cron 演示
thread = await client.threads.create(
    metadata={
        "__name__": "第21节课：Cron API测试"
    }
)
thread_id = thread["thread_id"]
```

## 创建 Cron

### 创建无状态 Cron

> POST /runs/crons

无状态 Cron 每次执行时都会**自动创建一个新的线程**。用 `on_run_completed` 控制线程去留：
- `delete`（默认）：运行完成后删除线程，无任何残留
- `keep`：保留线程，会不断积累线程数据，需要自己清理

> 为了不在课堂上真的频繁触发任务，这里先创建 `enabled=False` 的禁用 Cron，稍后再演示启用：

```python
# 创建无状态 Cron（每次执行都会新建线程）
cron = await client.crons.create(
    assistant_id=assistant_id,
    schedule="*/2 * * * *",  # 每 2 分钟执行一次
    input={"messages": [{"role": "user", "content": "定时任务：汇报一下当前时间"}]},
    metadata={"lesson": "第21节课：无状态Cron测试"},
    enabled=False,  # 先禁用，避免课堂演示时频繁触发
)
cron
```

`*/2 * * * *` 是标准的 **cron 表达式**，由 5 个字段组成：

| 位置 | 字段 | 取值范围 | 示例说明 |
| :--- | :--- | :--- | :--- |
| 1 | 分钟 | 0-59 | `*/2` 表示每 2 分钟 |
| 2 | 小时 | 0-23 | `9` 表示早上 9 点 |
| 3 | 日 | 1-31 | `*` 表示每天 |
| 4 | 月 | 1-12 | `*` 表示每月 |
| 5 | 星期 | 0-6（0 为周日） | `0` 表示周日 |

常用示例：
- `0 9 * * *`：每天早上 9 点
- `*/5 * * * *`：每 5 分钟
- `0 0 * * 1`：每周一零点

不传 `timezone` 时按 **UTC** 时区解释。

### 创建线程 Cron（有状态）

> POST /threads/{thread_id}/runs/crons

与无状态 Cron 的区别：通过 `create_for_thread` 绑定一个固定的线程。每次执行都往**同一个线程**里追加消息，可以积累多轮对话的上下文：

```python
# 创建绑定线程的 Cron（每次执行都复用同一个线程）
thread_cron = await client.crons.create_for_thread(
    thread_id=thread_id,
    assistant_id=assistant_id,
    schedule="0 9 * * *",  # 每天早上 9 点
    input={"messages": [{"role": "user", "content": "每日晨报"}]},
    metadata={"lesson": "第21节课：线程Cron测试"},
    enabled=False,  # 先禁用
)
thread_cron_id = thread_cron["cron_id"] # type: ignore
thread_cron
```

### cron 对象字段说明

- `cron_id`：Cron 任务 ID
- `thread_id`：绑定的线程（无状态 Cron 会为每次执行新建，通常为空）
- `assistant_id`：执行时使用的助手
- `schedule`：cron 表达式
- `payload`：每次执行时要创建运行的参数（input、config 等）
- `next_run_date`：下一次执行时间
- `end_time`：Cron 的结束时间（为空则无限期运行）
- `enabled`：是否启用
- `metadata`：创建时传入的元数据，可用于搜索过滤

## Cron 管理

### 搜索 Cron

> POST /runs/crons/search

列出 / 搜索全部 Cron 任务，支持按 `assistant_id`、`thread_id`、`enabled`、`metadata` 精确过滤，并支持分页与排序：

```python
# 按 metadata 精确过滤
crons = await client.crons.search(
    metadata={"lesson": "第21节课：无状态Cron测试"},
)
crons
```

```python
# 列出全部 Cron（限制 10 条）
all_crons = await client.crons.search(limit=10)
[(c["cron_id"], c["schedule"], c["enabled"], c["next_run_date"]) for c in all_crons]
```

### 根据 ID 获取 Cron

> GET /runs/crons/{cron_id}

根据 `cron_id` 精确获取**单个** Cron 任务。

> 当前 `langgraph-sdk==0.4.2` 的 `client.crons` 尚未封装该接口（只有 `search` 按条件过滤、`count` 统计），需要借助底层 HTTP 客户端 `client.http.get` 直接调用：

```python
# 根据 ID 获取单个 Cron
cron_detail = await client.http.get(f"/runs/crons/{thread_cron_id}")
cron_detail
```

### 统计 Cron 数量

> POST /runs/crons/count

按条件统计 Cron 的数量：

```python
# 统计全部 Cron 数量
count = await client.crons.count()
count
```

### 启用 / 禁用 Cron

> PATCH /runs/crons/{cron_id}

通过 `update(enabled=...)` 开关 Cron。这里把刚才禁用的无状态 Cron 启用，观察 `next_run_date` 开始生效：

```python
# 启用无状态 Cron
cron = await client.crons.update(
    cron_id=cron["cron_id"], # type: ignore
    enabled=True,
)
cron["enabled"], cron["next_run_date"]
```

```python
# 再次禁用，防止课堂期间真的执行
cron = await client.crons.update(
    cron_id=cron["cron_id"],
    enabled=False,
)
cron["enabled"]
```

### 修改 Cron

> PATCH /runs/crons/{cron_id}

`update` 同样可以修改调度计划、输入、元数据等：

```python
# 修改线程 Cron 的调度计划
updated = await client.crons.update(
    cron_id=thread_cron_id,
    schedule="30 8 * * *",
    input={"messages": [{"role": "user", "content": "每日晨报（提前半小时）"}]},
)
updated["schedule"], updated["payload"]["input"]
```

### 删除 Cron

> DELETE /runs/crons/{cron_id}

删除 Cron 任务，删除后再次查询会抛出 `NotFoundError`：

```python
crons = await client.crons.search()
crons
```

```python
from langgraph_sdk.errors import NotFoundError

for cron in crons:
    # 删除线程 Cron
    await client.crons.delete(cron["cron_id"])
    print(f'已删除:{cron["cron_id"]}')

```
