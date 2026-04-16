# framework/api/response — ApiBody / ApiResponseCode

## ApiBody

모든 API 응답의 body 클래스. `adapter/input/rest/common/` 에서 typealias 로 감싸 사용.

```kotlin
@JsonPropertyOrder("success", "code", "message", "payload")
data class ApiBody<T>(
    val success: Boolean,
    val code: String? = null,
    val message: String,
    val payload: T? = null,
) {
    companion object {
        // ApiAdvice, Security Handler 등 framework 레이어 전용
        fun <T> ofDTO(
            success: Boolean,
            apiCode: ApiResponseCode,
            payload: T? = null,
        ): ApiBody<T> =
            ApiBody(success, apiCode.code, apiCode.message, payload)

        fun <T> clientError(
            apiCode: ApiResponseCode,
            payload: T? = null,
        ): ResponseEntity<ApiBody<T>> =
            ResponseEntity.status(apiCode.status).body(
                ApiBody(false, apiCode.code, apiCode.message, payload)
            )

        fun <T> serverError(
            apiCode: ApiResponseCode = ApiResponseCode.SEVER_UNHANDLED_EXCEPTION,
            payload: T? = null,
        ): ResponseEntity<ApiBody<T>> =
            ResponseEntity.status(apiCode.status).body(
                ApiBody(false, apiCode.code, apiCode.message, payload)
            )
    }
}
```

> Controller 반환 타입은 `ApiResponse<T>` (typealias) 사용 — `adapter/input/rest/common/common.md` 참조
> Framework 레이어(ApiAdvice, Security Handler)에서는 `ApiBody.clientError()`, `ApiBody.serverError()` 사용

---

## ApiResponseCode

```kotlin
enum class ApiResponseCode(
    val code: String,
    val status: HttpStatus,
    val message: String,
) {
    /* SUCCESS */
    OK("S001", HttpStatus.OK, "리소스 요청에 성공했습니다."),
    CREATED("S002", HttpStatus.CREATED, "리소스 생성 요청에 성공했습니다."),
    UPDATED("S003", HttpStatus.OK, "리소스 수정 요청에 성공했습니다."),
    DELETED("S004", HttpStatus.OK, "리소스 삭제 요청에 성공했습니다."),
    SUCCESS_CREDENTIALS("S005", HttpStatus.OK, "인증에 성공했습니다."),

    /* REQUEST — CC */
    REQUEST_INVALID("CC001", HttpStatus.BAD_REQUEST, "유효성 검증에 실패했습니다."),
    REQUEST_INVALID_DATA("CC002", HttpStatus.BAD_REQUEST, "유효하지 않은 데이터입니다."),
    REQUEST_INVALID_BODY("CC003", HttpStatus.BAD_REQUEST, "요청 바디가 잘못되었습니다."),
    REQUEST_MISSING_HEADER("CC004", HttpStatus.BAD_REQUEST, "필수 헤더가 누락되었습니다."),
    REQUEST_UNSUPPORTED_METHOD("CC005", HttpStatus.BAD_REQUEST, "지원하지 않는 HTTP 메서드입니다."),
    REQUEST_UNSUPPORTED_REQUEST("CC006", HttpStatus.BAD_REQUEST, "지원하지 않는 요청입니다."),

    /* SECURITY — SC */
    INVALID_CREDENTIALS("SC001", HttpStatus.UNAUTHORIZED, "인증에 실패했습니다."),
    AUTHENTICATION_REQUIRED("SC002", HttpStatus.UNAUTHORIZED, "인증되지 않았습니다."),
    ACCESS_DENIED("SC003", HttpStatus.FORBIDDEN, "권한이 없습니다."),
    ACCOUNT_LOCKED("SC004", HttpStatus.LOCKED, "로그인 시도 횟수를 초과하여 계정이 잠겼습니다."),
    INVALID_REFRESH_TOKEN("SC005", HttpStatus.UNAUTHORIZED, "유효하지 않은 RefreshToken입니다."),

    /* WEB CLIENT — RC */
    WEB_CLIENT_IS_4XX_ERROR("RC001", HttpStatus.BAD_REQUEST, "서버에서 외부 API 호출에 실패했습니다."),
    WEB_CLIENT_IS_5XX_ERROR("RC002", HttpStatus.INTERNAL_SERVER_ERROR, "서버에서 외부 API 호출에 실패했습니다."),

    /* SERVER — SE */
    SEVER_SQL_EXCEPTION("SE001", HttpStatus.INTERNAL_SERVER_ERROR, "데이터베이스 SQL 에러가 발생했습니다."),
    SEVER_DATABASE_EXCEPTION("SE002", HttpStatus.INTERNAL_SERVER_ERROR, "데이터베이스 에러가 발생했습니다."),
    SEVER_UNHANDLED_EXCEPTION("SE998", HttpStatus.INTERNAL_SERVER_ERROR, "서버에서 에러가 발생했습니다."),
    SEVER_CRITICAL_EXCEPTION("SE999", HttpStatus.INTERNAL_SERVER_ERROR, "서버에서 에러가 발생했습니다."),

    /* 도메인별 코드 추가 — 아래에 선언 */
    ;

    companion object {
        fun findByCode(code: String): ApiResponseCode? =
            entries.firstOrNull { it.code == code }
    }
}
```

## 코드 Prefix 규칙

| Prefix | 용도 | HTTP 상태 |
|---|---|---|
| `S0XX` | 성공 | 200, 201 |
| `CC0XX` | 클라이언트 요청 오류 | 400 |
| `SC0XX` | 보안/인증 | 401, 403, 423 |
| `RC0XX` | 외부 WebClient 오류 | 400, 500 |
| `SE0XX` | 서버 내부 오류 | 500 |
| `{도메인}0XX` | 도메인 전용 | 도메인별 정의 |

## 규칙

- `ApiBody` 필드 순서 고정 — `@JsonPropertyOrder("success", "code", "message", "payload")`
- 공통 prefix(`S`, `CC`, `SC`, `RC`, `SE`)는 변경 금지
- 도메인별 코드는 도메인 약어 prefix + 3자리 숫자 — 예: `FOO001`
- **prefix + 번호 중복 절대 금지** — 전체 `ApiResponseCode` 내에서 `code` 값은 유일해야 함
- 새 코드 추가 전 기존 코드 목록 확인 필수
