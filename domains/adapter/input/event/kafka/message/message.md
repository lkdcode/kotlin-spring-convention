# adapter/input/event/kafka/message — *TopicEvent.kt

Kafka Listener 가 수신하는 메시지 값 객체.
JSON 역직렬화 대상이므로 필드는 직렬화 가능한 원시 타입으로 선언.

```kotlin
data class FooTopicEvent(
    val id: Long,
    val roomId: String,
    val userId: Long,
    val content: String,
    val createdAt: Instant,
) {
    // 도메인 VO 변환 — 직렬화 제외
    @get:JsonIgnore
    val roomIdVo get() = FooRoomId.of(roomId)

    // application model 변환
    fun toModel() = CreateFooModel(
        id = id,
        content = content,
    )
}
```

## 규칙

- 클래스명: `*TopicEvent`
- `data class` — 불변 값 객체
- 필드 타입: `String`, `Long`, `Int`, `Instant` 등 직렬화 가능한 원시 타입
- 도메인 VO(`FooRoomId` 등) 를 필드로 직접 선언 금지 — 역직렬화 실패
- 도메인 VO 접근이 필요하면 `@get:JsonIgnore` computed property 로 변환
- application model 변환은 `toModel()` 로 처리
