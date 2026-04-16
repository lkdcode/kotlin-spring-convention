# adapter/output/producer/kafka — *ProducerAdapter.kt

포트: `application/ports/output/producer/`
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

## 규칙

- Topic 상수는 `KafkaTopic` 에 선언
- 비즈니스 로직 작성 금지 — `kafkaTemplate.send()` 호출만
- 포트 인터페이스에만 의존 (`rules/dependency.md` 참조)
