# framework/security/handler — 예외 핸들러

## ApiResponseWriter

Security 필터 체인 내 JSON 응답 작성 유틸리티.
Spring MVC ExceptionHandler 가 아닌 필터 레벨에서 응답을 직접 씁니다.

```kotlin
@Component
class ApiResponseWriter(
    private val objectMapper: ObjectMapper,
) {
    fun <T> writeResponse(
        response: HttpServletResponse?,
        success: Boolean,
        apiResponseCode: ApiResponseCode,
        payload: T? = null,
    ) {
        response?.contentType = "application/json"
        response?.characterEncoding = "UTF-8"
        response?.status = apiResponseCode.status.value()
        response?.writer?.write(
            objectMapper.writeValueAsString(
                ApiResponse.ofDTO(
                    success = success,
                    apiCode = apiResponseCode,
                    payload = payload,
                )
            )
        )
        response?.writer?.flush()
        response?.flushBuffer()
    }
}
```

> 필터에서는 `@ControllerAdvice` 가 동작하지 않으므로 직접 `HttpServletResponse` 에 씁니다.

---

## ApiAuthenticationEntryPoint (401)

인증되지 않은 요청 처리.

```kotlin
@Component
class ApiAuthenticationEntryPoint(
    private val apiResponseWriter: ApiResponseWriter,
) : AuthenticationEntryPoint {

    override fun commence(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authException: AuthenticationException,
    ) {
        apiResponseWriter.writeResponse<Unit>(
            response = response,
            success = false,
            apiResponseCode = ApiResponseCode.UNAUTHORIZED,
        )
    }
}
```

## ApiAccessDeniedHandler (403)

인증은 됐으나 권한이 없는 요청 처리.

```kotlin
@Component
class ApiAccessDeniedHandler(
    private val apiResponseWriter: ApiResponseWriter,
) : AccessDeniedHandler {

    override fun handle(
        request: HttpServletRequest,
        response: HttpServletResponse,
        accessDeniedException: AccessDeniedException,
    ) {
        apiResponseWriter.writeResponse<Unit>(
            response = response,
            success = false,
            apiResponseCode = ApiResponseCode.FORBIDDEN,
        )
    }
}
```

## CorsSecurity

```kotlin
@Component
class CorsSecurity : CorsConfigurationSource {

    override fun getCorsConfiguration(
        request: HttpServletRequest,
    ): CorsConfiguration =
        CorsConfiguration().apply {
            allowedOriginPatterns = AllowedOrigin.PATTERNS
            allowedHeaders = listOf("*")
            allowedMethods = listOf("HEAD", "GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH")
            exposedHeaders = listOf("Set-Cookie", TOKEN_HEADER_KEY, REFRESH_HEADER_KEY)
            allowCredentials = true
        }
}

object AllowedOrigin {
    val PATTERNS = listOf(
        "http://localhost:*",
        "http://127.0.0.1:*",
    )
}
```

## 규칙

- 필터 레벨 응답은 반드시 `ApiResponseWriter.writeResponse()` 사용 — 직접 `response.writer.write()` 금지
- `ApiResponseWriter` 는 `@Component` — DI 로 주입
- CORS `exposedHeaders` 에 `Authorization`, `Refresh-Token` 둘 다 포함
- 운영 환경 `AllowedOrigin` 은 `@Profile` 로 환경별 분리
