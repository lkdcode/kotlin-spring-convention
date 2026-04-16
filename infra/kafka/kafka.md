# infra/kafka — Kafka

Spring Kafka. Consumer / Producer 설정 분리.
수신: `domains/adapter/input/event/kafka/` / 발행: `domains/adapter/output/producer/kafka/`

## 설정

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
```

## KafkaTopic

토픽 이름 enum 으로 중앙 관리.

```kotlin
enum class KafkaTopic {
    FOO_TOPIC,
    BAR_TOPIC,
    ;

    companion object {
        const val FOO_TOPIC_STRING = "FOO_TOPIC"
        const val BAR_TOPIC_STRING = "BAR_TOPIC"
    }
}
```

## KafkaConsumerConfig

`latest` / `earliest` 두 가지 팩토리 제공. 수동 ack(`MANUAL`) 사용.

```kotlin
@EnableKafka
@Configuration
class KafkaConsumerConfig(
    @Value("\${spring.kafka.bootstrap-servers}") private val bootstrapServers: String,
    private val objectMapper: ObjectMapper,
) {
    @Bean(LATEST_KAFKA_LISTENER_CONTAINER_FACTORY)
    fun latestKafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, String> =
        ConcurrentKafkaListenerContainerFactory<String, String>().apply {
            consumerFactory = consumerFactory("latest")
            containerProperties.ackMode = ContainerProperties.AckMode.MANUAL
            setCommonErrorHandler(errorHandler())
            setRecordMessageConverter(StringJsonMessageConverter(objectMapper))
        }

    @Bean(EARLIEST_KAFKA_LISTENER_CONTAINER_FACTORY)
    fun earliestKafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, String> =
        ConcurrentKafkaListenerContainerFactory<String, String>().apply {
            consumerFactory = consumerFactory("earliest")
            containerProperties.ackMode = ContainerProperties.AckMode.MANUAL
            setCommonErrorHandler(errorHandler())
            setRecordMessageConverter(StringJsonMessageConverter(objectMapper))
        }

    private fun consumerFactory(offsetReset: String): ConsumerFactory<String, String> =
        DefaultKafkaConsumerFactory(
            mapOf(
                ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
                ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to offsetReset,
                ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG to false,
                ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
                ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
                ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG to 10000,
                ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG to 30000,
            )
        )

    private fun errorHandler(): CommonErrorHandler =
        DefaultErrorHandler(FixedBackOff(1000L, 3))

    companion object {
        const val LATEST_KAFKA_LISTENER_CONTAINER_FACTORY = "latestKafkaListenerContainerFactory"
        const val EARLIEST_KAFKA_LISTENER_CONTAINER_FACTORY = "earliestKafkaListenerContainerFactory"
    }
}
```

## KafkaProducerConfig

```kotlin
@EnableKafka
@Configuration
class KafkaProducerConfig(
    @Value("\${spring.kafka.bootstrap-servers}") private val bootstrapServers: String,
) {
    @Bean
    fun kafkaTemplate(): KafkaTemplate<String, String> =
        KafkaTemplate(
            DefaultKafkaProducerFactory(
                mapOf(
                    ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
                    ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
                    ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
                    ProducerConfig.ACKS_CONFIG to "all",
                    ProducerConfig.RETRIES_CONFIG to 3,
                    ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true,
                )
            )
        )
}
```

## KafkaEventModel + KafkaProducer

발행 시 `KafkaEventModel` 로 토픽/키/페이로드를 묶어 전달.

```kotlin
class KafkaEventModel(
    val topic: KafkaTopic,
    val key: String,
    val payload: Any?,
) {
    val topicStringValue get() = topic.name

    companion object {
        fun of(topic: KafkaTopic, key: String, payload: Any? = null) =
            KafkaEventModel(topic, key, payload)
    }
}

@Component
class KafkaProducer(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val objectMapper: ObjectMapper,
) {
    fun publish(model: KafkaEventModel) {
        val json = objectMapper.writeValueAsString(model.payload)
        kafkaTemplate.send(model.topicStringValue, model.key, json)
    }
}
```

## 토픽 파티션 설정

```kotlin
@Configuration
class KafkaPartitionConfig {
    @Bean
    fun fooTopic(): NewTopic =
        TopicBuilder.name(KafkaTopic.FOO_TOPIC.name)
            .partitions(3)
            .replicas(1)
            .build()
}
```

## 규칙

- 토픽 이름은 `KafkaTopic` enum 으로만 관리 — 문자열 직접 사용 금지
- Consumer 는 수동 ack — `ack.acknowledge()` 는 try/catch 밖에서 호출
- Producer 는 `KafkaProducer` 를 통해서만 발행 — `KafkaTemplate` 직접 사용 금지
- `latest` / `earliest` 선택 기준: `adapter/input/event/kafka/kafka.md` 참조
