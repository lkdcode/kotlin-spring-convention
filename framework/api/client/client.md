# framework/api/client — ApiWebClient

외부 API 호출 전용 WebClient 래퍼. 4xx/5xx 에러를 `ApiException` 으로 변환.

## ApiWebClient (snake_case 응답)

```kotlin
@Service
class ApiWebClient(
    private val webClient: WebClient,
) {
    fun <T> getRequest(
        url: String,
        headers: Consumer<HttpHeaders>,
        responseType: ParameterizedTypeReference<T>,
    ): T? =
        webClient.get()
            .uri(url)
            .headers(headers)
            .accept(MediaType.APPLICATION_JSON)
            .retrieve()
            .onStatus({ it.is4xxClientError }) {
                it.bodyToMono(String::class.java).map { body ->
                    logInfo("${ApiResponseCode.WEB_CLIENT_IS_4XX_ERROR.message}: $body")
                    throw ApiException(ApiResponseCode.WEB_CLIENT_IS_4XX_ERROR)
                }
            }
            .onStatus({ it.is5xxServerError }) {
                it.bodyToMono(String::class.java).map { body ->
                    logInfo("${ApiResponseCode.WEB_CLIENT_IS_5XX_ERROR.message}: $body")
                    throw ApiException(ApiResponseCode.WEB_CLIENT_IS_5XX_ERROR)
                }
            }
            .bodyToMono(responseType)
            .block()

    fun <T> getRequest(
        url: String,
        headers: Consumer<HttpHeaders>,
        responseType: Class<T>,
    ): T? = /* 동일한 구조 */

    fun <BODY, T> postRequest(
        url: String,
        headers: Consumer<HttpHeaders>,
        body: BODY,
        responseType: Class<T>,
    ): T? =
        webClient.post()
            .uri(url)
            .headers(headers)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(body as Any)
            .retrieve()
            .onStatus({ it.is4xxClientError }) { ... }
            .onStatus({ it.is5xxServerError }) { ... }
            .bodyToMono(responseType)
            .block()
}
```

## ApiWebClientCamelCase (camelCase 응답)

외부 API 응답이 camelCase 일 때 사용. `@Qualifier("camelCaseWebClient")` 로 별도 ObjectMapper 주입.

```kotlin
@Service
class ApiWebClientCamelCase(
    @Qualifier("camelCaseWebClient") private val webClient: WebClient,
) {
    // ApiWebClient 와 동일한 메서드 구조
    // camelCase ObjectMapper 를 사용하는 WebClient 빈 주입
}
```

## WebClient 빈 설정

```kotlin
@Configuration
class WebClientConfig(
    private val objectMapper: ObjectMapper,
) {
    @Bean
    fun webClient(): WebClient =
        WebClient.builder()
            .codecs { it.defaultCodecs().jackson2JsonDecoder(Jackson2JsonDecoder(objectMapper)) }
            .build()

    // camelCase 전용 — 외부 API 응답이 camelCase 인 경우
    @Bean("camelCaseWebClient")
    fun camelCaseWebClient(): WebClient =
        WebClient.builder()
            .codecs {
                it.defaultCodecs().jackson2JsonDecoder(
                    Jackson2JsonDecoder(ObjectMapper().apply { propertyNamingStrategy = PropertyNamingStrategies.LOWER_CAMEL_CASE })
                )
            }
            .build()
}
```

## 규칙

- 외부 API 호출은 반드시 `ApiWebClient` 사용 — 직접 `WebClient` 사용 금지
- 4xx → `WEB_CLIENT_IS_4XX_ERROR`, 5xx → `WEB_CLIENT_IS_5XX_ERROR`
- 기본 응답: snake_case → `ApiWebClient`
- camelCase 응답: `ApiWebClientCamelCase`
- `.block()` 사용 — Spring MVC + Virtual Thread 환경에서 블로킹 허용
