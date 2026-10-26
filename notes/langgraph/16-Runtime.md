# Runtime

`LangGraph`提供了两种注入自定义配置的渠道：

1. 通过`config.configurable`注入，通过`config.configurable`读取
2. 通过`context`注入，通过`runtime.context`读取

目前的情况是：

- `LangGraph`官方建议使用`context`进行自定义配置
- `config.configurable`仅用于配置线程id
- `LangGraph`仍然支持两种自定义配置的方式
- 如果使用`Agent Server`，两者不能混用，建议使用`context`

最佳实践 —— 用一个就行，别两个一起用：
- 新项目一律使用`context`
- 旧项目用的是哪个，就继续用哪个，不要混

## 注入和使用 Context

```python
from pydantic import BaseModel, Field
from typing_extensions import TypedDict
from langgraph.runtime import Runtime, get_runtime
from langgraph.config import get_config
from langgraph.graph import END, START, StateGraph


# 1. 定义 ContextSchema：本次运行允许传入哪些配置（必须有默认值或必传字段）
class ContextSchema(BaseModel):
    user_id: str = Field(description="用户id")
    locale: str = Field(description="语言偏好", default="zh-CN")



# 2. 定义图状态
class State(TypedDict):
    pass

# 3. 在节点中读取 Context
def greet(state: State, runtime: Runtime[ContextSchema]):
    user_id = runtime.context.user_id
    locale = runtime.context.locale
    runtime = get_runtime()
    config = get_config()
    return {}

# 4. 创建图时传入 context_schema
builder = StateGraph(State, context_schema=ContextSchema)
builder.add_node(greet)
builder.add_edge(START, "greet")
builder.add_edge("greet", END)

graph = builder.compile()

# 5. 调用时传入 context
await graph.ainvoke(
    input={},
    context=ContextSchema(user_id="张三"),
)
```

## 修改自定义配置为 context

- 在`states/core_agent_state.py`中添加`ContextSchema`
- 在`graphs/core_agent_graph.py`中注入`context_schema`
- 在`call_model`中消费`context`

```python
from langgraph_python.graphs.core_agent_graph import build_graph
from langchain.messages import HumanMessage

graph = build_graph().compile()
```

```python
from langgraph_python.states.core_agent_state import ContextSchema

await graph.ainvoke(
    input={
        "messages": [HumanMessage("你好，你是谁？")]
    },
    context=ContextSchema(model="fake"),
)
```

## 核心理解

Assistants = 图 + 配置
