# framework/storage/local — 내부 저장소

서버 로컬 파일시스템에 저장. Java NIO 기반.

## 설정

```yaml
# application.yml
storage:
  prefix: http://localhost:8080/storage/   # 파일 접근 URL prefix
  path: /files/storage/                    # 실제 저장 경로 (절대경로)
```

## StorageProperties

```kotlin
@Component
data class StorageProperties(
    @Value("\${storage.prefix}")
    val prefix: String,

    @Value("\${storage.path}")
    val path: String,
) {
    fun locations() = "file:$path"
}
```

## StorageService

```kotlin
@Service
class StorageService(
    private val properties: StorageProperties,
    private val storagePathUtil: StoragePathUtil,
) {
    // 저장 — prefix + path + filename 반환
    fun save(file: MultipartFile, prefix: String = ""): String {
        val filename = StringUtils.cleanPath(file.originalFilename ?: UUID.randomUUID().toString())
        val storageLocation: Path = Paths.get(properties.path + prefix).toAbsolutePath().normalize()

        try {
            Files.createDirectories(storageLocation)
            val targetLocation = storageLocation.resolve(filename)
            Files.copy(file.inputStream, targetLocation, StandardCopyOption.REPLACE_EXISTING)
            return "${properties.prefix}${prefix}$filename"
        } catch (ex: IOException) {
            logInfo(ex.message)
            throw ApiException(ApiResponseCode.FILE_CREATE_FAIL)
        }
    }

    // 수정 — 기존 파일 삭제 후 새 파일 저장
    fun update(path: String, newFile: MultipartFile): String {
        delete(path)
        return save(newFile)
    }

    // 삭제 — 파일 + 빈 디렉토리 정리
    fun delete(path: String) {
        val removedPrefix = path.removePrefix(properties.prefix + "/")
        val fullPath = properties.path + removedPrefix

        try {
            val targetPath = Paths.get(fullPath)
            if (Files.deleteIfExists(targetPath)) {
                val parentPath = targetPath.parent
                if (parentPath != null
                    && Files.isDirectory(parentPath)
                    && Files.list(parentPath).use { it.count() == 0L }
                ) {
                    Files.deleteIfExists(parentPath)
                }
            }
        } catch (ex: IOException) {
            logInfo("${ApiResponseCode.FILE_DELETE_FAIL.message}: $path")
        }
    }

    // 도메인별 경로 위임
    fun getFooPath() = storagePathUtil.getFooPath()
    fun getFooPrefix() = storagePathUtil.getFooPrefix()
}
```

## StoragePathUtil

도메인별 저장 경로 생성. `{domain}/{year}/{month}/{day}/{UUID}/` 패턴.

```kotlin
@Service
class StoragePathUtil(
    private val properties: StorageProperties,
) {
    // 경로 생성 — 도메인별 메서드
    fun getFooPath() = "${FOO}/${year()}/${month()}/${day()}/${UUID.randomUUID()}/"

    // prefix — 조회 시 URL prefix
    fun getFooPrefix() = "${properties.prefix}${FOO}"

    companion object {
        const val FOO = "/foo"   // 도메인별 상수 정의

        fun year(): String = String.format("%d", TimeUtil.nowKST().year)
        fun month(): String = String.format("%02d", TimeUtil.nowKST().month.value)
        fun day(): String = String.format("%02d", TimeUtil.nowKST().dayOfMonth)
    }
}
```

## 규칙

- `StorageService` 를 통해서만 파일 저장/삭제 — 직접 `Files.*` 호출 금지
- 파일 삭제 실패는 `logInfo` 로만 처리 — 비즈니스 실패로 간주하지 않음
- 빈 디렉토리 자동 정리 — 파일 삭제 후 부모 디렉토리가 비어있으면 함께 삭제
- 도메인 경로 상수는 `StoragePathUtil.companion` 에 선언
