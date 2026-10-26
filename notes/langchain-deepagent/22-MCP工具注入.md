# MCP工具注入

```python
!uv add langchain-mcp-adapters
```

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    connections={
        "langchain_mcp": {
            "transport": "streamable_http",
            "url": "https://docs.langchain.com/mcp",
        }
    },
    tool_name_prefix=True
)
tools = await client.get_tools()
```

```python
tools
```
