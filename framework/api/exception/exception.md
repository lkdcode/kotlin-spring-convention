# framework/api/exception — ApiException

비즈니스 예외를 `ApiResponseCode` 로 표현하는 커스텀 예외.

```kotlin
class ApiException(
    private val response: ApiResponseCode,
    val payload: Any? = null,
) : RuntimeException(response.message) {

    fun name(): String = response.name
    fun code(): String = response.code
    fun status(): HttpStatus = response.status
    fun message(): String = response.message
    fun getApiResponseCode(): ApiResponseCode = response
}
```

## 사용 예시

```kotlin
// 단순 throw
throw ApiException(ApiResponseCode.FOO_NOT_FOUND)

// payload 포함 (디버그 정보, 복원 가능 데이터 등)
throw ApiException(ApiResponseCode.FOO_DUPLICATE_CODE, "중복된 코드: $code")

// Validator 에서 사용
override fun validateDuplicate(code: String) {
    if (exists) throw ApiException(ApiResponseCode.FOO_DUPLICATE_CODE)
}
```

## 규칙

- `RuntimeException` 상속 — `@Transactional` 롤백 자동 적용
- `payload` 는 클라이언트에 전달할 부가 정보 (선택)
- 도메인 예외(`DomainException`)와 구분 — `ApiException` 은 application/adapter 계층에서만 사용
- `DomainException` → `ApiAdvice` 에서 `ApiException` 으로 변환
