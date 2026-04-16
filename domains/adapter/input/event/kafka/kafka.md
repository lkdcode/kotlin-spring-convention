# adapter/input/event/kafka — *EventListener.kt

## 디렉토리 구조

```
kafka/
├── *EventListener.kt     ← @KafkaListener — 수신만 담당
├── message/
│   └── *TopicEvent.kt    ← Kafka 메시지 DTO
├── handler/
│   └── *Handler.kt       ← 이벤트별 처리 로직 (UseCase 호출)
└── property/
    └── Kafka*GroupIdProperty.kt  ← Group ID 상수
```

---

## EventListener — *EventListener.kt

수신만 담당. 처리는 Handler 에 위임. `ack.acknowledge()` 는 항상 호출.

```kotlin
@Component
class FooEventListener(
    private val fooCreateHandler: FooCreateHandler,
    private val fooUpdateHandler: FooUpdateHandler,
) {
    @KafkaListener(
        topics = [KafkaTopic.FOO_TOPIC_STRING],
        groupId = KafkaFooGroupIdProperty.FOO_CREATE,
        containerFactory = LATEST_KAFKA_LISTENER_CONTAINER_FACTORY,
        concurrency = "3",
    )
    fun handleCreate(
        event: FooTopicEvent,
        ack: Acknowledgment,
    ) {
        try {
            fooCreateHandler.handle(event)
        } catch (e: Exception) {
            logWarn("🛑 FOO CREATE 실패\n${e.stackTraceToString()}")
        }

        ack.acknowledge()
    }

    @KafkaListener(
        topics = [KafkaTopic.FOO_TOPIC_STRING],
        groupId = KafkaFooGroupIdProperty.FOO_UPDATE,
        containerFactory = LATEST_KAFKA_LISTENER_CONTAINER_FACTORY,
        concurrency = "3",
    )
    fun handleUpdate(
        event: FooTopicEvent,
        ack: Acknowledgment,
    ) {
        try {
            fooUpdateHandler.handle(event)
        } catch (e: Exception) {
            logWarn("🛑 FOO UPDATE 실패\n${e.stackTraceToString()}")
        }

        ack.acknowledge()
    }
}
```

---

## Handler — handler/*Handler.kt

UseCase 호출 전 가벼운 분기 처리 담당.

```kotlin
@Component
class FooCreateHandler(
    private val createUsecase: CreateFooUsecase,
) {
    fun handle(event: FooTopicEvent) {
        createUsecase.execute(event.toModel())
    }
}
```

조건 분기가 필요한 경우:

```kotlin
@Component
class FooNotificationHandler(
    private val notifyUsecase: NotifyFooUsecase,
) {
    fun handle(event: FooTopicEvent) {
        if (event.type == FooType.SYSTEM) return   // 조건부 스킵

        notifyUsecase.execute(event.toModel())
    }
}
```

---

## Event DTO — event/*TopicEvent.kt

Kafka 메시지 역직렬화 전용. `toModel()` 로 application model 변환.

```kotlin
data class FooTopicEvent(
    val id: Long,
    val name: String,
    val type: String,
) {
    fun toModel() = CreateFooModel(
        id = id,
        name = name,
    )
}
```

---

## Group ID 상수 — property/Kafka*GroupIdProperty.kt

```kotlin
class KafkaFooGroupIdProperty {
    companion object {
        const val FOO_CREATE = "FOO_CREATE_GROUP"
        const val FOO_UPDATE = "FOO_UPDATE_GROUP"
    }
}
```

---

## containerFactory 선택 기준

| | `EARLIEST` | `LATEST` |
|---|---|---|
| 오프셋 | 처음부터 읽음 | 새 메시지만 읽음 |
| 사용 | 이력 데이터 처리, 재처리 필요 | 실시간 처리 |

---

## 공통 규칙 (event 계층 전체)

- 비즈니스 로직 직접 작성 금지
- UseCase(interface) 만 호출 — event 전용 UseCase 가 없으면 생성
- 트랜잭션은 UseCase 에서만 처리
- 포트 인터페이스에만 의존 — 구현체 직접 import 금지 (`rules/dependency.md` 참조)

## Kafka 규칙

- 클래스명: `*EventListener`
- `ack.acknowledge()` 는 try/catch 밖 — 예외 발생해도 **항상** 호출
- Listener 는 수신만 — 처리 로직은 Handler 에 위임(delegate)
- Topic 상수는 `KafkaTopic` 에, Group ID 상수는 `Kafka*GroupIdProperty` 에 선언
- **멱등성 보장** — 중복 메시지 처리 고려
