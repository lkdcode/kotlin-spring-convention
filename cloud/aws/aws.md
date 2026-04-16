# cloud/aws — Amazon Web Services

## 설정

```yaml
aws:
  app:
    access_key_id: your-access-key
    secret_access_key: your-secret-key
    bucket_name: your-bucket
    region: ap-northeast-2
  presigned:
    get_expired: {사용자 정의}    # GET URL 만료 (초)
    put_expired: {사용자 정의}    # PUT URL 만료 (초)
```

## 서비스 목록

| 디렉토리 | 서비스 |
|---|---|
| `s3/` | Simple Storage Service — Pre-signed URL |
