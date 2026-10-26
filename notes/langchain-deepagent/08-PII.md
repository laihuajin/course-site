# PII

本节学习如何使用 `PIIMiddleware` 检测并处理 Agent 中的个人身份信息。

## 什么是 PII

PII 是 **Personally Identifiable Information** 的缩写，即个人身份信息。它是能够直接或间接识别某个人的信息，例如：

- 姓名、身份证号、学号
- 手机号、邮箱、家庭住址
- 银行卡号、信用卡号
- IP 地址、设备地址

## PIIMiddleware

`PIIMiddleware` 是 LangChain 提供的预置中间件。它使用确定性的检测规则找到消息中的 PII，再按照指定策略处理。

### 内置检查类型

| 类型 | 检测内容 |
| --- | --- |
| `email` | 邮箱地址 |
| `credit_card` | 通过 Luhn 校验的信用卡号 |
| `ip` | IPv4 地址 |
| `mac_address` | MAC 地址 |
| `url` | URL |

### 处理策略

| 策略 | 作用 | 示例 |
| --- | --- | --- |
| `redact` | 使用占位符替换 | `[REDACTED_EMAIL]` |
| `mask` | 隐藏一部分，只保留末尾字符 | `**** **** **** 1111` |
| `hash` | 替换为稳定的哈希值 | `<ip_hash:805ebf20>` |
| `block` | 抛出异常并终止 Run | `PIIDetectionError` |

### 拦截位置

`PIIMiddleware` 使用的是**节点钩子**，不是 `wrap_model_call` 或 `wrap_tool_call`。

```text
  ...
   ↓
before_model  ← 检查点
   ↓
model
   ↓
after_model   ← 检查点
   ↓
  ...
```

`before_model` 和 `after_model` 都可能在一次 Run 中执行多次，因此模型输出检查针对的是**每次模型调用产生的 `AIMessage`**，而不只是 Run 的最终输出。

| 参数 | 默认值 | 检查对象 | 执行位置 |
| --- | --- | --- | --- |
| `apply_to_input` | `True` | 最近一条 `HumanMessage` | `before_model` |
| `apply_to_output` | `False` | 最近一条 `AIMessage` | `after_model` |
| `apply_to_tool_results` | `False` | 工具产生的 `ToolMessage` | 下一次调用模型前的 `before_model` |

检测到 PII 后，中间件会创建处理后的新消息，并返回 `{'messages': new_messages}` 更新图状态。因此它修改的不只是临时模型请求，最终图状态中的对应消息也会改变。

> `block` 是例外：它会抛出异常并终止 Run，不会把脱敏消息写入状态。

## 测试准备

```python
from langgraph_sdk import get_client

client = get_client(url="http://127.0.0.1:2024")
```

```python
async def run_agent(graph_id: str, content: str):
    # 删除所有线程，免得看晕了
    threads = await client.threads.search(limit=100)
    for t in threads:
        await client.threads.delete(thread_id=t['thread_id'])

    # 创建新线程
    thread = await client.threads.create(metadata={"__name__": "PII测试"})
    thread_id = thread["thread_id"]

    assistants = await client.assistants.search(graph_id=graph_id)
    assistant_id = assistants[0]["assistant_id"]
    
    return await client.runs.wait(
        thread_id=thread_id,
        assistant_id=assistant_id,
        input={
            "messages": [
                {"role": "user", "content": content}
            ]
        },
    )
```

## 拦截模型输入

`pii_input` 使用以下配置：

```python
middleware=[
    PIIMiddleware("email", strategy="redact"),
    PIIMiddleware("credit_card", strategy="mask"),
    PIIMiddleware("ip", strategy="hash"),
    PIIMiddleware("mac_address", strategy="redact"),
    PIIMiddleware("url", strategy="redact"),
]
```

```python
await run_agent(
    "pii_input",
    (
        "邮箱 zhangsan@example.com，"
        "卡号 4111 1111 1111 1111，"
        "IP 192.168.1.10，"
        "MAC 00:1A:2B:3C:4D:5E，"
        "网址 https://example.com/profile"
    ),
)
```

## 阻止包含 PII 的请求

`pii_block` 使用以下配置：

```python
middleware=[
    PIIMiddleware("email", strategy="block"),
]
```

`block` 不会继续调用模型，而是让 Run 失败。

```python
try:
    await run_agent("pii_block", "我的邮箱是 zhangsan@example.com")
except Exception as error:
    print(type(error).__name__)
    print(error)
```

## 自定义 PII

`pii_custom` 使用以下配置：

```python
middleware=[
    PIIMiddleware(
        "student_id",
        detector=r"STU-\d{6}",
        strategy="redact",
    ),
]
```

内置类型不能覆盖所有业务数据。例如，系统可能需要把学号当作 PII。可以通过 `detector` 提供正则表达式。

`student_id` 是自定义类型名称，替换后的占位符为 `[REDACTED_STUDENT_ID]`。

```python
await run_agent("pii_custom", "我的学号是 STU-123456")
```

## 自定义检测函数

`pii_custom_detector` 使用以下配置：

```python
def detect_phone_number(content: str):
    matches = []
    for match in re.finditer(r"(?<!\d)1[3-9]\d{9}(?!\d)", content):
        matches.append(
            {
                "value": match.group(),
                "start": match.start(),
                "end": match.end(),
            }
        )
    return matches


middleware=[
    PIIMiddleware(
        "phone_number",
        detector=detect_phone_number,
        strategy="mask",
    ),
]
```

检测函数接收待检查的字符串，返回匹配内容及其起止位置。

```python
await run_agent("pii_custom_detector", "我的手机号是 13812345678")
```

## 拦截模型输出

`pii_output` 使用以下配置：

```python
middleware=[
    PIIMiddleware(
        "email",
        strategy="redact",
        apply_to_input=False,
        apply_to_output=True,
    ),
]
```

Fake 模型会固定返回 `support@example.com`。中间件会在每次模型调用后的 `after_model` 中处理 `AIMessage`。

```python
await run_agent("pii_output", "请提供联系方式")
```

## 拦截工具结果

`pii_tool_result` 使用以下配置：

```python
middleware=[
    PIIMiddleware(
        "email",
        strategy="redact",
        apply_to_input=False,
        apply_to_output=False,
        apply_to_tool_results=True,
    ),
]
```


```python
await run_agent("pii_tool_result", "查询客户资料")
```
