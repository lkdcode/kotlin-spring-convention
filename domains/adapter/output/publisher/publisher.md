# adapter/output/publisher — 브로드캐스트 어댑터

도메인 이벤트 / 메시지를 외부 채널로 발행하는 아웃바운드 어댑터.

## 디렉토리 구조

```
adapter/output/
├── publisher/                         → ports/output/publisher
│   ├── socket/*PublisherAdapter.kt  ← WebSocket broadcast
│   └── spring/*PublisherAdapter.kt  ← Spring ApplicationEvent
├── producer/                        → ports/output/producer
│   └── kafka/*ProducerAdapter.kt    ← Kafka 이벤트 발행
└── notifier/                        → ports/output/notifier
    └── fcm/*NotifierAdapter.kt      ← FCM Push 알림
```

---

## Socket — *PublisherAdapter.kt

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

---

## Spring — *PublisherAdapter.kt

Spring ApplicationEventPublisher 로 도메인 이벤트 발행.
`@TransactionalEventListener` 핸들러가 수신.

```kotlin
@Component
class SpringDomainEventPublisherAdapter(
    private val applicationEventPublisher: ApplicationEventPublisher,
) : DomainEventPublisher {

    override fun publish(
        event: DomainEvent,
    ) {
        applicationEventPublisher.publishEvent(event)
    }
}
```

---

## Kafka — *ProducerAdapter.kt

Kafka 토픽으로 이벤트 발행.

```kotlin
@Component
class FooProducerAdapter(
    private val kafkaTemplate: KafkaTemplate<String, Any>,
) : FooProducer {

    override fun produce(
        key: String,
        message: FooMessage,
    ) {
        kafkaTemplate.send(KafkaTopic.FOO_TOPIC, key, message)
    }
}
```

---

## FCM — *NotifierAdapter.kt

디바이스 토큰으로 푸시 알림 발송.

```kotlin
@Component
class FooNotifierAdapter(
    private val fcmService: FcmService,
) : FooNotifier {

    override fun notify(
        deviceTokenList: List<String>,
        model: FooModel,
    ) {
        fcmService.pushDataOnly(
            deviceTokenList,
            FcmPushModel.of(ChannelType.FOO, OperationType.PUB, model),
        )
    }

    override fun notify(
        deviceToken: String,
        model: FooModel,
    ) {
        fcmService.pushDataOnly(
            deviceToken,
            FcmPushModel.of(ChannelType.FOO, OperationType.PUB, model),
        )
    }
}
```

---

## 네이밍

| 채널 | 어댑터명 | 어댑터 위치 | 포트 위치 |
|---|---|---|---|
| WebSocket | `*PublisherAdapter` | `adapter/output/publisher/socket/` | `ports/output/publisher/` |
| Spring Event | `*PublisherAdapter` | `adapter/output/publisher/spring/` | `ports/output/publisher/` |
| Kafka | `*ProducerAdapter` | `adapter/output/producer/kafka/` | `ports/output/producer/` |
| FCM | `*NotifierAdapter` | `adapter/output/notifier/fcm/` | `ports/output/notifier/` |

---

## 규칙

- 채널별 어댑터 분리 — 같은 클래스에 혼용 금지
- Socket 어댑터: `convert()` private helper 로 직렬화 로직 분리
- 비즈니스 로직 작성 금지 — 변환 + 발행만 담당
- 포트 인터페이스에만 의존 (`rules/dependency.md` 참조)
