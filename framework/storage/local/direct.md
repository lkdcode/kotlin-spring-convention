# framework/storage/local/direct — Direct 저장

Client 가 multipart 로 서버에 직접 전송. 서버가 로컬 파일시스템에 저장.

## 흐름

```
Client --[POST multipart]--> Server --저장--> Local FileSystem
                         <-- 파일 URL 반환 --
```

## 설정

```yaml
storage:
  prefix: http://localhost:8080/storage/
  path: /files/storage/
```

## StorageProperties

```kotlin
@Component
data class StorageProperties(
    @Value("\${storage.prefix}") val prefix: String,
    @Value("\${storage.path}") val path: String,
) {
    fun locations() = "file:$path"
}
```

## LocalDirectStorageAdapter

```kotlin
@Service
class LocalDirectStorageAdapter(
    private val properties: StorageProperties,
) : DirectStoragePort {

    override fun save(
        file: MultipartFile,
        path: String,
    ): String {
        val filename = StringUtils.cleanPath(
            file.originalFilename ?: UUID.randomUUID().toString()
        )
        val storageLocation = Paths.get(properties.path + path).toAbsolutePath().normalize()

        try {
            Files.createDirectories(storageLocation)
            Files.copy(file.inputStream, storageLocation.resolve(filename), StandardCopyOption.REPLACE_EXISTING)
            return "${properties.prefix}${path}$filename"
        } catch (ex: IOException) {
            logInfo(ex.message)
            throw ApiException(ApiResponseCode.FILE_CREATE_FAIL)
        }
    }

    override fun delete(key: String) {
        val fullPath = properties.path + key.removePrefix(properties.prefix)

        try {
            val targetPath = Paths.get(fullPath)
            if (Files.deleteIfExists(targetPath)) {
                val parent = targetPath.parent
                if (parent != null
                    && Files.isDirectory(parent)
                    && Files.list(parent).use { it.count() == 0L }
                ) {
                    Files.deleteIfExists(parent)
                }
            }
        } catch (ex: IOException) {
            logInfo("${ApiResponseCode.FILE_DELETE_FAIL.message}: $key")
        }
    }
}
```

## Controller 예시

```kotlin
@PostMapping("/api/foos/files")
fun upload(
    @RequestParam file: MultipartFile,
): ApiResponse<String> {
    val path = storagePathUtil.getFooPath()
    val url = directStoragePort.save(file, path)

    return ApiResponse(url)
}
```

## 규칙

- 파일 삭제 실패는 `logInfo` 로만 처리 — 비즈니스 실패로 간주하지 않음
- 빈 디렉토리 자동 정리 — 파일 삭제 후 부모 디렉토리가 비면 함께 삭제
- 저장 경로는 `StoragePathUtil.getFooPath()` 등 도메인별 메서드 사용
