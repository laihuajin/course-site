# 搭建MinIO环境

## 安装环境

1. 覆盖 `docker-compose.yml` 文件
2. 运行 `docker compose up -d shared-minio` 启动服务

## 生成专用凭据

### 安装 MinIO Client（mc）

`mc` 是 MinIO 官方的命令行管理工具，独立于 Python 的 `minio` SDK。

[MinIO Client 下载页面](https://min.io/download#/windows) 下载

### 保存管理员连接

```bash
mc alias set shared http://localhost:9002 root 12345678
```

| 参数 | 含义 |
|---|---|
| `mc alias set` | 保存或更新本机连接配置 |
| `shared` | 自定义连接别名，后续命令使用它找到连接 |
| `http://localhost:9002` | Shared MinIO 的对象 API 地址 |
| `root` | 当前 Compose 中的管理员 Access Key |
| `12345678` | 当前 Compose 中的管理员 Secret Key |

### 准备专用权限策略

将 `resource/minio-policy.json` 复制到你的工程

### 自动生成专用凭据

```text
mc admin user svcacct add \
  --policy resource/minio-policy.json \
  shared root
```

1. 将凭证保存到环境变量
2. 新增`minio_settings.py`



### 查看和撤销专用凭据

由管理员列出 root 下的服务账号：

```bash
mc admin user svcacct ls shared root
```

如果需要删除凭据，将下面的占位值替换为实际 Access Key：

```bash
mc admin user svcacct rm shared '<实际生成的 Access Key>'
```
