# System API

Agent Server 还提供了一组 **System API**，用于**健康检查**、**服务器信息** 与 **监控指标**，方便部署、监控与运维。

API文档地址：http://localhost:2024/docs#tag/System

API调用：推荐 `langgraph_sdk`，但当前版本（`0.4.2`）**尚未封装 System 接口**，需要借助底层 HTTP 客户端 `client.http` 直接调用

System 接口共 4 个：
- **`GET /ok`**：健康检查，可选是否检查数据库连通性
- **`GET /info`**：服务器版本、功能开关与元数据
- **`GET /metrics`**：Prometheus / JSON 格式的系统监控指标
- **`GET /docs`**：API 文档页面（HTML）

## 安装 LangGraph SDK

上一节课已经安装过 `langgraph-sdk`，这里重复执行也无副作用，仅保证课件可独立运行：

```python
!uv add langgraph-sdk==0.4.2
```

## 创建客户端

连接本地 Agent Server（默认端口 2024）。System 接口没有独立的子客户端，全部通过 `client.http` 调用：

```python
from langgraph_sdk import get_client

# 连接本地 Agent Server
client = get_client(url="http://localhost:2024")
client
```

## 健康检查 `GET /ok`

检查服务器是否健康，返回 `{"ok": true}`。`check_db` 参数可选，传 `1` 时同时检查数据库连通性：

```python
# 基本健康检查
await client.http.get("/ok")
```

```python
# 同时检查数据库连通性
await client.http.get("/ok", params={"check_db": 1})
```

> 服务器异常时（或数据库连不上且 `check_db=1`）会返回 `500`，`client.http` 会自动抛出 `ServerError`，适合在部署脚本 / 负载均衡的健康检查探针中使用。

## 服务器信息 `GET /info`

返回服务器版本、`langgraph` 库版本、功能开关（`flags`）与部署元数据（`metadata`）：

```python
info = await client.http.get("/info")
info
```

`info` 字段说明：

- `version`：LangGraph API 服务器版本
- `langgraph_py_version`：`langgraph` Python 库版本
- `flags`：已启用的功能特性开关
- `metadata`：服务器部署元数据

```python
# 分别查看各字段
print("服务器版本:", info["version"])
print("langgraph Python 版本:", info["langgraph_py_version"])
print("功能开关:", info["flags"])
print("部署元数据:", info["metadata"])
```

## 系统指标 `GET /metrics`

获取系统监控指标，支持两种输出格式（`format` 参数）：
- `prometheus`（默认）：Prometheus 文本格式，适合接入 Prometheus 采集器
- `json`：JSON 格式，包含队列统计、Worker 统计、HTTP 统计等，适合程序化处理

### JSON 格式

`client.http.get` 内部会把响应体按 **JSON** 解析，因此 JSON 格式可以直接使用：

```python
metrics = await client.http.get("/metrics", params={"format": "json"})
metrics
```

```python
# 查看队列与 Worker 统计
metrics
```

### Prometheus 格式

默认返回 `text/plain` 文本，`client.http.get` 会按 JSON 解析而报错。需要借助底层的 `httpx.AsyncClient`（即 `client.http.client`）获取原始响应文本：

```python
# Prometheus 格式（text/plain），使用原始 httpx 客户端获取
r = await client.http.client.get(
    "/metrics",
    params={"format": "prometheus"},
)
print("Content-Type:", r.headers["content-type"])
print(r.text[:1000])
```

### 为什么 Prometheus 格式不能用 `client.http.get`？

`client.http.get` 在拿到响应后，会把响应体交给 `orjson.loads` 解析成 Python 对象。Prometheus 文本格式不是 JSON，解析必然失败。所以凡是返回**非 JSON** 内容的接口（`/metrics` 的 Prometheus 格式、`/docs` 的 HTML），都要改用原始 `httpx` 客户端。

## API 文档 `GET /docs`

返回 Swagger / OpenAPI 交互式文档页面（HTML）。
