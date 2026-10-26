# Mock Model

某些场景的测试需要使用模拟的模型，比如：报错、死循环、工具调用错误、...

## langchain_core 中的模拟模型

| 模拟模型 | 用途 |
| --- | --- |
| `FakeChatModel` | 最简单的模拟聊天模型，无论输入什么消息，都固定返回 `fake response` 字符串，用于最基本的单元测试 |
| `FakeListChatModel` | 模拟聊天模型，内部维护一个字符串列表 `responses`，每次调用依次（循环）返回列表中的一条作为回复，也支持流式输出 |
| `FakeMessagesListChatModel` | 模拟聊天模型，内部维护一个 `BaseMessage` 列表 `responses`，每次调用依次（循环）返回列表中的一条消息，可以返回带工具调用等结构的消息 |
| `GenericFakeChatModel` | 通用模拟聊天模型，接收一个消息迭代器 `messages`，每次调用取出一条（`AIMessage` 或字符串）作为回复，支持同步、异步与流式测试 |
| `ParrotFakeChatModel` | 鹦鹉学舌式模拟聊天模型，会把输入消息列表中的最后一条消息原样返回，用于测试消息回显逻辑 |
| `FakeListLLM` | 模拟普通 LLM（文本补全模型），内部维护一个字符串列表 `responses`，每次调用依次（循环）返回一条，用于非聊天类模型的测试 |
| `FakeStreamingListLLM` | 模拟支持流式输出的 LLM，继承自 `FakeListLLM`，除了依次返回列表中的回复外，还支持按流式块返回，并可模拟在指定块位置抛错 |

## 使用封装好的 mock model

- `mock_model`: `core/mock_model.py`
- `mock_model 配置`: `core/mock_model.config.jonc`

```python
from langgraph_python.core.mock_model import mock_model
from langchain.messages import HumanMessage, AIMessage, ToolMessage

model  = mock_model()
```

```python
await model.ainvoke([
    HumanMessage("你好！"),
    ToolMessage(content="当前时间:xxxx", tool_call_id="ab")
])
```

```python
chunks = model.astream([
    HumanMessage("你好!")
])
async for chunk in chunks:
    for block in chunk.content_blocks:
        print(block)
```
