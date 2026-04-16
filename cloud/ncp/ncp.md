# infra/ncp — Naver Cloud Platform

## 설정

```yaml
ncp:
  app:
    url: https://sens.apigw.ntruss.com
    sms_service_id: your-service-id
    access_key: your-access-key
    secret_key: your-secret-key
    time_stamp_header_key: x-ncp-apigw-timestamp
    access_key_header_key: x-ncp-iam-access-key
    signature_header_key: x-ncp-apigw-signature-v2
```

## 서비스 목록

| 디렉토리 | 서비스 |
|---|---|
| `sms/` | SMS / LMS 발송 (SENS) |
