# adapter/output/publisher/socket — *PublisherAdapter.kt

포트: `application/ports/output/publisher/`
WebSocket 토픽으로 브로드캐스트. `convert()` 로 직렬화 관심사 분리.

```kotlin
@Component
class FooPublisherAdapter(
    private val socketService: SocketService,
    private val objectMapper: ObjectMapper,
) : FooPublisher {

    override fun publish(
        roomId: FooRoomId,
        model: FooModel,
    ) {
        val json = convert(OperationType.PUB, model)
        socketService.toTopic(roomId.channelCode, json)
    }

    override fun publishWithoutMe(
        userId: Long,
        roomId: FooRoomId,
        model: FooModel,
    ) {
        val json = convert(OperationType.PUB, model)
        socketService.toTopicWithoutMe(roomId.channelCode, json, userId)
    }

    private fun convert(
        operationType: OperationType,
        response: Any,
    ): String =
        objectMapper.writeValueAsString(
            SocketResponse.broadcastResponse(operationType, ChannelType.FOO, response)
        )
}
```

## 규칙

- `convert()` private helper 필수 — 직렬화 로직 분리
- 비즈니스 로직 작성 금지 — 변환 + 발행만
- 포트 인터페이스에만 의존 (`rules/dependency.md` 참조)
