# cloud/ncp/sms — NCP SMS (SENS)

NCP SENS API 를 통한 SMS/LMS 발송.
인증은 HMAC-SHA256 서명 방식.

## NcpProperty

```kotlin
@Component
data class NcpProperty(
    @Value("\${ncp.app.url}") val url: String,
    @Value("\${ncp.app.sms_service_id}") val smsServiceId: String,
    @Value("\${ncp.app.access_key}") val accessKey: String,
    @Value("\${ncp.app.secret_key}") val secretKey: String,
    @Value("\${ncp.app.time_stamp_header_key}") val timeStampHeaderKey: String,
    @Value("\${ncp.app.access_key_header_key}") val accessKeyHeaderKey: String,
    @Value("\${ncp.app.signature_header_key}") val signatureHeaderKey: String,
) {
    val timeStampNow: String get() = Instant.now().toEpochMilli().toString()
    val sendMessageEndPoint: String get() = "/sms/v2/services/$smsServiceId/messages"
    val sendMessageApi: String get() = "$url$sendMessageEndPoint"
}
```

## BaseNcpSmsApiService

HMAC-SHA256 서명 생성 + HTTP 요청 처리.

```kotlin
@Service
class BaseNcpSmsApiService(
    private val property: NcpProperty,
    private val webClient: ApiWebClient,
) {
    val timeStampNow: String get() = Instant.now().toEpochMilli().toString()

    fun <T> postRequest(
        url: String,
        body: Any?,
        response: Class<T>,
        timestamp: String,
        signature: String,
    ): Mono<T> = webClient.postRequest(url, createHeader(timestamp, signature), body, response)

    fun makeSignature(method: String, url: String, timestamp: String): String {
        val message = "$method $url\n$timestamp\n${property.accessKey}"
        val signingKey = SecretKeySpec(property.secretKey.toByteArray(Charsets.UTF_8), ALGORITHM)
        val mac = Mac.getInstance(ALGORITHM).apply { init(signingKey) }
        return Base64.getEncoder().encodeToString(mac.doFinal(message.toByteArray(Charsets.UTF_8)))
    }

    private fun createHeader(timestamp: String, signature: String): Consumer<HttpHeaders> =
        Consumer { headers ->
            headers.add(property.timeStampHeaderKey, timestamp)
            headers.add(property.accessKeyHeaderKey, property.accessKey)
            headers.add(property.signatureHeaderKey, signature)
        }

    companion object {
        private const val ALGORITHM = "HmacSHA256"
    }
}
```

## NcpSmsSendRequestModel / NcpSmsSendResponseDTO

```kotlin
interface NcpSmsSendRequestModel {
    data class MessageModel(
        val type: String? = "LMS",
        val contentType: String? = "COMM",
        val countryCode: String? = "82",
        val from: String,
        val subject: String? = null,
        val content: String,
        val messages: List<MessageContentModel> = listOf(),
    )

    data class MessageContentModel(
        val to: String,
        val subject: String? = null,
        val content: String? = null,
    )
}

interface NcpSmsSendResponseDTO {
    data class ResultDTO(
        val requestId: String,
        val requestTime: String,
        val statusCode: String,
        val statusName: String,
    )
}
```

## NcpSmsSendService

`SmsSendService` 포트를 구현. `framework/` 의 포트 인터페이스를 통해 도메인에 노출.

```kotlin
@Service
class NcpSmsSendService(
    private val service: BaseNcpSmsApiService,
    private val property: NcpProperty,
) : SmsSendService {

    override fun send(messageModel: NcpSmsSendRequestModel.MessageModel): Mono<Void> {
        val timestamp = service.timeStampNow
        val signature = service.makeSignature(POST, property.sendMessageEndPoint, timestamp)

        return service
            .postRequest(property.sendMessageApi, messageModel, NcpSmsSendResponseDTO.ResultDTO::class.java, timestamp, signature)
            .then()
    }

    companion object {
        private const val POST = "POST"
    }
}
```

## 규칙

- `NcpSmsSendService` 는 `SmsSendService` 포트 구현 — `domains/` 에서 직접 import 금지
- HMAC 서명은 `BaseNcpSmsApiService.makeSignature()` 로 통일
- timestamp 는 매 요청마다 새로 생성 — 재사용 금지
