# Agent 接入 MCP

<img src="./assets/mcp.svg">

to do list:

- `mcp` 独立配置
- 进一步封装描述和工具
- 注册工具
- 修改系统提示词模板和注入

```python
from agent.agent import Agent
from agent.printer import ModelPrinterListener

listener = ModelPrinterListener()
agent = Agent(model_listener=listener)
```

```python
agent.session.print_friendly()
```

```python
agent.invoke("当前时间是？")
```

```python
agent.session.save()
```
