# framework/storage/port — 저장소 포트 인터페이스

## DirectStoragePort

서버가 파일을 직접 받아 저장.

```kotlin
interface DirectStoragePort {
    fun save(file: MultipartFile, path: String): String   // 저장 후 접근 URL 반환
    fun delete(key: String)
}
```

## PresignedStoragePort

URL 을 발급하고 클라이언트가 직접 업로드. Local Pre-signed, S3 모두 이 포트를 구현.

```kotlin
interface PresignedStoragePort {
    fun putUrl(key: String): URL           // 업로드용 URL 발급
    fun getUrl(key: String): URL?          // 다운로드용 URL 발급 (없으면 null)
    fun delete(key: String)
    fun copy(sourceKey: String, destKey: String)
}
```

## 구현체 매핑

| 인터페이스 | 구현체 | 저장 위치 |
|---|---|---|
| `DirectStoragePort` | `LocalDirectStorageAdapter` | 로컬 파일시스템 |
| `PresignedStoragePort` | `LocalPresignedStorageAdapter` | 로컬 파일시스템 |
| `PresignedStoragePort` | `AwsS3StorageAdapter` | AWS S3 |

## 규칙

- Application 계층은 포트 인터페이스에만 의존 — 구현체 직접 import 금지
- 프로젝트 내에서 저장 방식 하나를 선택해 일관성 유지
- `Direct` ↔ `Presigned` 혼용 금지 — 혼용 시 클라이언트 코드 이중화
