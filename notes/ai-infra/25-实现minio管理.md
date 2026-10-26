```python
from deal_pilot.utils.minio import MinIO
from deal_pilot.config.minio_settings import minio_settings
minio = MinIO(
    endpoint=minio_settings.endpoint,
    access_key=minio_settings.access_key,
    secret_key=minio_settings.secret_key,
    secure=minio_settings.secure
)
```

```python
bucket_name = "agent-artifacts"
```

```python
await minio.create_bucket(bucket_name)
```

```python
await minio.upload_dir(
    bucket_name=bucket_name, 
    source_dir="/Volumes/数据盘/课程资料/AI/ai-infrastructure/src",
    prefix="/uploads"
)
```

```python
minio.get_file_url(bucket_name=bucket_name, filename="uploads/deal_pilot/__init__.py")
```

```python
await minio.remove_bucket(bucket_name, True)
```
