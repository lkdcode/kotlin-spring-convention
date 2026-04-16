# framework/api/advice — ApiAdvice

모든 예외를 중앙에서 처리. `ApiBody` 형식으로 통일된 응답 반환.

```kotlin
@RestControllerAdvice
class ApiAdvice {

    // 비즈니스 예외 — ApiResponseCode 기반
    @ExceptionHandler(ApiException::class)
    fun handleApiException(e: ApiException): ResponseEntity<ApiBody<Any>> {
        logInfo("ApiResponseCode: ${e.getApiResponseCode()}, Message: ${e.message()} Status: ${e.status()}")
        return ApiBody.clientError(e.getApiResponseCode(), e.payload)
    }

    // @Valid 실패 — 필드 단위 오류 목록 반환
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleMethodArgumentNotValidException(e: MethodArgumentNotValidException): ResponseEntity<ApiBody<List<*>>> {
        val response = e.bindingResult.fieldErrors.map {
            logInfo("Field: ${it.field}, Rejected value: ${it.rejectedValue}, Message: ${it.defaultMessage}")
            mapOf("Field" to it.field, "Rejected value" to it.rejectedValue, "Message" to it.defaultMessage)
        }
        return ApiBody.clientError(ApiResponseCode.REQUEST_INVALID, response)
    }

    // @Validated 실패
    @ExceptionHandler(ConstraintViolationException::class)
    fun handleConstraintViolationException(e: ConstraintViolationException): ResponseEntity<ApiBody<List<*>>> {
        val response = e.constraintViolations.map {
            mapOf("Invalid value" to it.propertyPath.toString(), "Message" to it.message)
        }
        return ApiBody.clientError(ApiResponseCode.REQUEST_INVALID_DATA, response)
    }

    // 요청 바디 파싱 실패 (JSON 역직렬화 오류 포함)
    @ExceptionHandler(HttpMessageNotReadableException::class)
    fun handleHttpMessageNotReadableException(e: HttpMessageNotReadableException): ResponseEntity<ApiBody<String>> {
        logInfo("HttpMessageNotReadableException: ${e.message}")
        return ApiBody.clientError(ApiResponseCode.REQUEST_INVALID_BODY, "${e.message}")
    }

    // DateTime 포맷 오류 — 예: LocalDate 에 "2024-01-01T00:00:00" 전달
    @ExceptionHandler(DateTimeParseException::class)
    fun handleDateTimeParseException(e: DateTimeParseException): ResponseEntity<ApiBody<String>> {
        logInfo("DateTimeParseException: '${e.parsedString}' — ${e.message}")
        return ApiBody.clientError(
            apiCode = ApiResponseCode.REQUEST_INVALID_DATA,
            payload = "날짜/시간 형식이 올바르지 않습니다: '${e.parsedString}' (예: yyyy-MM-dd)",
        )
    }

    // PathVariable, RequestParam 타입 불일치 — Long 에 문자열 전달, LocalDate 포맷 오류 등
    @ExceptionHandler(MethodArgumentTypeMismatchException::class)
    fun handleMethodArgumentTypeMismatchException(e: MethodArgumentTypeMismatchException): ResponseEntity<ApiBody<String>> {
        logInfo("MethodArgumentTypeMismatchException: param=${e.name}, value=${e.value}, type=${e.requiredType?.simpleName}")
        val message = when {
            e.requiredType?.isAssignableFrom(LocalDate::class.java) == true ->
                "날짜 형식이 올바르지 않습니다: '${e.value}' (예: yyyy-MM-dd)"
            e.requiredType?.isAssignableFrom(LocalDateTime::class.java) == true ->
                "날짜/시간 형식이 올바르지 않습니다: '${e.value}' (예: yyyy-MM-ddTHH:mm:ss)"
            else ->
                "파라미터 '${e.name}' 의 타입이 올바르지 않습니다: '${e.value}'"
        }
        return ApiBody.clientError(ApiResponseCode.REQUEST_INVALID_DATA, message)
    }

    // 필수 헤더 누락
    @ExceptionHandler(MissingRequestHeaderException::class)
    fun handleMissingRequestHeaderException(e: MissingRequestHeaderException): ResponseEntity<ApiBody<String>> {
        logInfo("MissingRequestHeaderException: ${e.body.detail}")
        return ApiBody.clientError(ApiResponseCode.REQUEST_MISSING_HEADER, "${e.body.detail}")
    }

    // 지원하지 않는 HTTP 메서드
    @ExceptionHandler(HttpRequestMethodNotSupportedException::class)
    fun handleHttpRequestMethodNotSupportedException(
        e: HttpRequestMethodNotSupportedException,
        request: HttpServletRequest,
    ): ResponseEntity<ApiBody<String>> {
        logInfo("Method: ${e.method} End-Point: ${request.requestURI}")
        return ApiBody.clientError(ApiResponseCode.REQUEST_UNSUPPORTED_METHOD, "${e.method} ${request.requestURI}")
    }

    // 지원하지 않는 Content-Type
    @ExceptionHandler(HttpMediaTypeNotSupportedException::class)
    fun handleHttpMediaTypeNotSupportedException(e: HttpMediaTypeNotSupportedException): ResponseEntity<ApiBody<String>> {
        logInfo("HttpMediaTypeNotSupportedException: ${e.contentType}")
        return ApiBody.clientError(
            apiCode = ApiResponseCode.REQUEST_INVALID_BODY,
            payload = "지원하지 않는 Content-Type 입니다: ${e.contentType}",
        )
    }

    // 파일 크기 초과
    @ExceptionHandler(MaxUploadSizeExceededException::class)
    fun handleMaxUploadSizeExceededException(e: MaxUploadSizeExceededException): ResponseEntity<ApiBody<String>> {
        logInfo("MaxUploadSizeExceededException: maxSize=${e.maxUploadSize}")
        return ApiBody.clientError(
            apiCode = ApiResponseCode.REQUEST_INVALID_DATA,
            payload = "파일 크기가 허용된 최대 크기를 초과했습니다.",
        )
    }

    // 존재하지 않는 엔드포인트
    @ExceptionHandler(NoResourceFoundException::class)
    fun handleNoResourceFoundException(e: NoResourceFoundException): ResponseEntity<ApiBody<String>> {
        logInfo("NoResourceFoundException: ${e.resourcePath}")
        return ApiBody.clientError(ApiResponseCode.REQUEST_UNSUPPORTED_REQUEST, "API: /${e.resourcePath}")
    }

    // DB SQL 예외
    @ExceptionHandler(SQLException::class)
    fun handleSQLException(e: SQLException): ResponseEntity<ApiBody<Unit>> {
        logWarn("SQLException: ${e.message}")
        return ApiBody.serverError(ApiResponseCode.SEVER_SQL_EXCEPTION)
    }

    // DB 접근 예외
    @ExceptionHandler(DataAccessException::class)
    fun handleDataAccessException(e: DataAccessException): ResponseEntity<ApiBody<Unit>> {
        logWarn("DataAccessException: ${e.message}")
        return ApiBody.serverError(ApiResponseCode.SEVER_SQL_EXCEPTION)
    }

    // 처리되지 않은 RuntimeException
    @ExceptionHandler(RuntimeException::class)
    fun handleRuntimeException(e: RuntimeException): ResponseEntity<ApiBody<Unit>> {
        logError("[Runtime Exception] ${e::class.simpleName}: ${e.message}\n${e.stackTraceToString()}", e)
        return ApiBody.serverError(ApiResponseCode.SEVER_UNHANDLED_EXCEPTION)
    }

    // 처리되지 않은 Exception
    @ExceptionHandler(Exception::class)
    fun handleException(e: Exception): ResponseEntity<ApiBody<Unit>> {
        logError("[Unhandled Exception] ${e::class.simpleName}: ${e.message}", e)
        return ApiBody.serverError(ApiResponseCode.SEVER_CRITICAL_EXCEPTION)
    }
}
```

## 예외 처리 우선순위

```
ApiException                         ← 비즈니스 예외 (최우선)
MethodArgumentNotValidException      ← @Valid 실패
ConstraintViolationException         ← @Validated 실패
HttpMessageNotReadableException      ← 바디 파싱/JSON 역직렬화 실패
DateTimeParseException               ← 날짜 포맷 오류 (yyyy-MM-dd 기대, ISO8601 수신 등)
MethodArgumentTypeMismatchException  ← PathVariable/RequestParam 타입 불일치
MissingRequestHeaderException        ← 필수 헤더 누락
HttpRequestMethodNotSupportedException
HttpMediaTypeNotSupportedException   ← Content-Type 오류
MaxUploadSizeExceededException       ← 파일 크기 초과
NoResourceFoundException             ← 존재하지 않는 엔드포인트
SQLException / DataAccessException   ← DB 예외
RuntimeException                     ← 처리 안 된 예외 (stackTrace 로깅)
Exception                            ← 최후 보루
```

## 로그 레벨 기준

| 예외 종류 | 로그 레벨 | 이유 |
|---|---|---|
| `ApiException` | `logInfo` | 예상된 비즈니스 오류 |
| 파라미터/포맷 오류 | `logInfo` | 클라이언트 실수 |
| `MaxUploadSizeExceededException` | `logInfo` | 클라이언트 실수 |
| `SQLException`, `DataAccessException` | `logWarn` | DB 이상 징후 |
| `RuntimeException` | `logError` + stackTrace | 예상치 못한 오류 |
| `Exception` | `logError` | 최후 보루 |

## 규칙

- `@RestControllerAdvice` 하나만 — 중복 선언 금지
- `ApiBody.clientError()` / `ApiBody.serverError()` 사용 — 직접 `ResponseEntity` 생성 금지
- `DateTimeParseException` 핸들러에서 예시 포맷 명시 — 클라이언트 디버깅 지원
- `MethodArgumentTypeMismatchException` 에서 타입별 메시지 분기 — LocalDate/LocalDateTime 별도 안내
