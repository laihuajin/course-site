```python
from deal_pilot.backends.sandbox_backend import SandboxBackend

backend = SandboxBackend(lambda:"abc")
```

```python
await backend.awrite("/root/workspace/1.txt", "hello")
```

```python
cmd = """(
cd /sandbox_utils && uv run python - <<'EOF'
from minio import MinIO

async def main():
    client = MinIO(
        endpoint="host.docker.internal:9002",
        access_key="0CM4SZV3Q0B9PLDFRE9S",
        secret_key="nkUyIN5GIkrCLYhbwuhBWG8ldIO+1l+Ay6Jcll0t",
        secure=False
    )
    bucket_name="agent-artifacts"
    await client.create_bucket(bucket_name)
    await client.upload_file(
        bucket_name=bucket_name,
        filename="/uploads/1.txt",
        source_file="/root/workspace/1.txt"
    )
import asyncio
asyncio.run(main())
print("上传完成")
EOF
)
"""

await backend.aexecute(
    command=cmd
)
```
