# Nacos自定义配置

在 `Nacos` 中添加自定义配置

- Group: deal_pilot
- Data ID: sandbox_config
- 配置格式：yaml
- 配置内容：

```yaml
# 创建沙箱使用的镜像，对应 Sandbox.create(image=...)。
image: "opensandbox-python-node:1.0"

# 沙箱创建后存活的时间，单位秒；对应 Sandbox.create(timeout=...)。
timeout_seconds: 600

# 等待沙箱创建并就绪的最长时间，单位秒；对应 Sandbox.create(ready_timeout=...)。
ready_timeout_seconds: 180

# 单次命令执行允许等待的最长时间，单位秒；对应 RunCommandOpts(timeout=...)。
command_timeout_seconds: 90
```

