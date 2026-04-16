# framework/storage/local/presigned — Local Pre-signed URL

S3 Pre-signed 와 동일한 인터페이스(`PresignedStoragePort`). 로컬 환경에서 S3 없이 동일한 흐름 테스트 가능.

## 흐름

```
1. Client --[POST /api/files/put-url?key=...]--> Server
                                             <-- 로컬 업로드 URL 반환

2. Client --[PUT {localUrl} + 파일]--> Server (업로드 엔드포인트)
                                   <-- 완료

3. Client --[POST /api/foos + key]--> Server (비즈니스 처리)
```

## 설정

```yaml
storage:
  prefix: http://localhost:8080/storage/
  path: /files/storage/
  upload-url: http://localhost:8080/storage/upload
  token-expired: 3600   # 업로드 URL 만료 (초)
```

## LocalPresignedStorageAdapter

```kotlin
@Service
class LocalPresignedStorageAdapter(
    private val properties: StorageProperties,
    private val cacheManager: CacheManager,
) : PresignedStoragePort {

    override fun putUrl(key: String): URL {
        val token = UUID.randomUUID().toString()
        cacheManager.getCache(UPLOAD_TOKEN_CACHE)?.put(token, key)
        return URL("${properties.uploadUrl}?token=$token")
    }

    companion object {
        const val UPLOAD_TOKEN_CACHE = "UPLOAD_TOKEN"
    }

    override fun getUrl(key: String): URL? {
        if (key.isBlank()) return null
        return URL("${properties.prefix}$key")
    }

    override fun delete(key: String) {
        // LocalDirectStorageAdapter.delete() 와 동일한 로직
    }

    override fun copy(sourceKey: String, destKey: String) {
        val src = Paths.get(properties.path + sourceKey)
        val dest = Paths.get(properties.path + destKey)
        Files.createDirectories(dest.parent)
        Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING)
    }
}
```

## 규칙

- S3 Pre-signed 와 완전히 동일한 `PresignedStoragePort` 구현 — 교체 시 코드 변경 없음
- 업로드 토큰 만료 관리 필수 — 미완료 업로드 임시파일 정리
- 로컬 개발 전용 — 운영 환경에서는 `AwsS3StorageAdapter` 로 교체
