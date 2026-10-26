# Nacos API 接口

## 前期准备

1. 进入 `Nacos` 后台，添加用户：

- 账号：deal_pilot
- 密码：自行约定

2. 为用户绑定角色

3. 赋予角色只读权限

4. 将`SKILL`和`Agent Spec`资源设置为公开

5. 按照 `.env.example` 添加新的环境变量

6. 将 `src/config/nacos_settings.py` 复制到你的工程

```python
!uv sync
```

## API接口文档

https://nacos.io/swagger/client/

## 登录并取得访问令牌

向默认鉴权插件发送 `POST /nacos/v3/auth/user/login`。

用户名、密码放在请求体中

```python
import httpx
from deal_pilot.config.nacos_settings import nacos_settings

async with httpx.AsyncClient(
    base_url=nacos_settings.server_address, 
    timeout=10
) as client:
    response = await client.post(
        "/nacos/v3/auth/user/login",
        data={
            "username": nacos_settings.username, 
            "password": nacos_settings.password
        },
    )
    response.raise_for_status()
    login_result = response.json()
access_token = login_result["accessToken"]
login_result
```

```python
from deal_pilot.config.nacos_config import get_access_token
access_token = await get_access_token()
access_token
```

可将该函数封装到 `config/nacos_config.py` 中

## 封装带令牌的异步 GET 请求

Nacos 的 JSON 接口返回 `code / message / data`。先检查 `code`，再取 `data`；Skill 下载接口返回 ZIP 字节流，不能直接按 JSON 解码。

```python
async def get_response(client: httpx.AsyncClient, path: str, params: dict[str, str]) -> httpx.Response:
    response = await client.get(path, params=params)
    response.raise_for_status()
    return response


async def get_json_data(client: httpx.AsyncClient, path: str, params: dict[str, str]):
    response = await get_response(client, path, params)
    result = response.json()
    if result.get("code") != 0:
        raise RuntimeError(f"Nacos 读取失败: {result.get('message', result)}")
    return result["data"]

```

## 读取 Skills

`GET /nacos/v3/client/ai/skills` 

```python
import io
import zipfile
from pathlib import Path
from rich import print

async with httpx.AsyncClient(
    base_url=nacos_settings.server_address, 
    headers={"accessToken": access_token}, 
    timeout=15
) as client:
    skill_name = "pptx"
    response = await get_response(
        client,
        "/nacos/v3/client/ai/skills",
        {
            "namespaceId": nacos_settings.namespace_id, 
            "name": skill_name,
            "label": "production"
        },
    )
    zip_bytes = response.content

# with zipfile.ZipFile(io.BytesIO(zip_bytes)) as archive:
    # print(archive.namelist())
    # bytes = archive.read("pptx/SKILL.md")
    # print(bytes.decode())
#     if not any(name == "SKILL.md" or name.endswith("/SKILL.md") for name in archive.namelist()):
#         raise RuntimeError(f"{skill_name} 的 ZIP 包中没有 SKILL.md")

project_root = next((path for path in (Path.cwd(), *Path.cwd().parents) if (path / "pyproject.toml").is_file()), None)
if project_root is None:
    raise RuntimeError("找不到当前工程的 pyproject.toml，无法保存 Skill ZIP")
zip_path = project_root / "temp" / "skills.zip"
zip_path.parent.mkdir(parents=True, exist_ok=True)
saved_bytes = zip_path.write_bytes(zip_bytes)

```

## 读取 MCP Server

本项目使用 Nacos **3.2.4**。该版本按名称读取 MCP 详情的 HTTP 接口是 `GET /nacos/v3/admin/ai/mcp`，因此只读用户需要目标命名空间的 `read` 权限。此接口不支持 `label` 参数，只能按 `version` 或默认最新版本读取，不能保证选中 `production` 标签。响应包含协议、版本、连接配置、端点和工具定义。下面显示课堂最关心的字段；完整响应保存在 `mcp_details_by_name`，供后续接入 MCP 客户端。

这里读取的是 **Nacos 中的注册信息**，不会与 MCP Server 建立连接，也不代表远端服务当前健康。

```python
import json

mcp_details_by_name = {}
async with httpx.AsyncClient(
    base_url=nacos_settings.server_address, headers={"accessToken": access_token}, timeout=15
) as client:
    for mcp_name in ("langchain-docs", "microsoft-docs"):
        details = await get_json_data(
            client,
            "/nacos/v3/admin/ai/mcp",
            {"namespaceId": nacos_settings.namespace_id, "mcpName": mcp_name},
        )
        mcp_details_by_name[mcp_name] = details
        tool_spec = details.get("toolSpec") or {}
        print(f"\n=== {mcp_name} ===")
        print("注册名称:", details.get("name"))
        print("协议:", details.get("protocol"))
        print("版本:", (details.get("versionDetail") or {}).get("version"))
        print("启用:", details.get("enabled"))
        print("端点:", json.dumps(details.get("backendEndpoints"), ensure_ascii=False, indent=2))
        print("工具:", [tool.get("name") for tool in tool_spec.get("tools", [])])

```

## 读取 Agent Spec

`GET /nacos/v3/client/ai/agentspecs` 使用 `name` 和 `label=production` 定位 Agent Spec。第 14 讲的 `main_agent`、`web_agent` 分别引用不同的 Skills 和 MCP；读取后对照控制台中的 `Agents.md`、`manifest.json` 内容。这里展示整个 `data`，并保留在 `agent_specs_by_name` 中。

Agent Spec 中的 Skill/MCP 名称是引用；取得 Agent Spec 不会自动下载 Skill 包、查询 MCP 详情或连接 MCP Server。

```python
import json

agent_specs_by_name = {}
async with httpx.AsyncClient(
    base_url=nacos_settings.server_address, headers={"accessToken": access_token}, timeout=15
) as client:
    for agent_name in ("main_agent", "web_agent"):
        spec = await get_json_data(
            client,
            "/nacos/v3/client/ai/agentspecs",
            {"namespaceId": nacos_settings.namespace_id, "name": agent_name, "label": "production"},
        )
        agent_specs_by_name[agent_name] = spec
        print(f"\n=== {agent_name} ===")
        print(json.dumps(spec, ensure_ascii=False, indent=2))

```
