# application/ports/output/producer — *Producer.kt (interface)

**이벤트 발행 전용** — 메시지 브로커로 이벤트 전파.
Kafka, RabbitMQ 등.
구현체: `adapter/output/producer/kafka/`

```kotlin
interface FooProducer {
    fun produce(key: String, message: FooMessage)
}
```

## 규칙

- 인터페이스에 kafka / rabbit 등 기술 용어 포함 금지
- 단일 책임 — 메시지 브로커 발행 역할만
- 브로드캐스트 → `publisher/`, 알림 → `notifier/` 로 분리
