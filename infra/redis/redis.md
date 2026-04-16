# infra/redis — Redis

Spring Data Redis + Lettuce. `framework/cache/remote/` 의 `RedisCacheService` 가 이 설정을 사용.

## 설정

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: {사용자 정의}
      database: 0
      ssl: false
```

## RedisProperties

```kotlin
@Component
data class RedisProperties(
    @Value("\${spring.data.redis.host}") val host: String,
    @Value("\${spring.data.redis.port}") val port: Int,
    @Value("\${spring.data.redis.password}") val password: String,
    @Value("\${spring.data.redis.database}") val database: Int,
    @Value("\${spring.data.redis.ssl}") val ssl: Boolean,
)
```

## RedisConfig

```kotlin
@Configuration
class RedisConfig(
    private val properties: RedisProperties,
    private val objectMapper: ObjectMapper,
) {
    @Bean
    fun redisConnectionFactory(): LettuceConnectionFactory {
        val standalone = RedisStandaloneConfiguration(properties.host, properties.port).apply {
            password = RedisPassword.of(properties.password)
            database = properties.database
        }

        val clientConfig = LettuceClientConfiguration.builder().apply {
            if (properties.ssl) useSsl()
            commandTimeout(Duration.ofSeconds(3))
            shutdownTimeout(Duration.ofMillis(100))
        }.build()

        return LettuceConnectionFactory(standalone, clientConfig)
    }

    @Bean
    fun redisTemplate(): RedisTemplate<String, Any> {
        val typeValidator = BasicPolymorphicTypeValidator.builder()
            .allowIfBaseType(Any::class.java)
            .build()

        val redisObjectMapper = objectMapper.copy().apply {
            registerModule(JavaTimeModule())
            disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)

            // Kotlin data class 다형성 문제 해결 — KotlinSupportingTypeResolver 사용
            val typeResolver = KotlinSupportingTypeResolver(
                ObjectMapper.DefaultTyping.NON_FINAL,
                typeValidator,
            )
            val initializedResolver = typeResolver
                .init(JsonTypeInfo.Id.CLASS, null)
                .inclusion(JsonTypeInfo.As.PROPERTY)

            setDefaultTyping(initializedResolver)
        }

        return RedisTemplate<String, Any>().apply {
            connectionFactory = redisConnectionFactory()
            keySerializer = StringRedisSerializer()
            valueSerializer = GenericJackson2JsonRedisSerializer(redisObjectMapper)
            hashKeySerializer = StringRedisSerializer()
            hashValueSerializer = GenericJackson2JsonRedisSerializer(redisObjectMapper)
        }
    }
}
```

## KotlinSupportingTypeResolver

Kotlin data class 를 Redis 에 저장할 때 `@class` 타입 정보가 누락되는 문제를 해결.

**문제:** 기본 `activateDefaultTyping` 은 Kotlin data class 의 non-final 특성을 제대로 처리하지 못해 역직렬화 시 타입 정보(`@class`) 가 없어 오류 발생.

```kotlin
class KotlinSupportingTypeResolver(
    defaultTyping: ObjectMapper.DefaultTyping,
    polymorphicTypeValidator: PolymorphicTypeValidator,
) : ObjectMapper.DefaultTypeResolverBuilder(defaultTyping, polymorphicTypeValidator) {

    // primitive, String 을 제외한 모든 타입에 @class 타입 정보 포함
    override fun useForType(javaType: JavaType): Boolean =
        !(javaType.isPrimitive || javaType.rawClass == String::class.java)
}
```

Redis 에 저장되는 JSON 예시:
```json
{
  "@class": "com.example.FooDTO",
  "id": 1,
  "name": "foo"
}
```

## 규칙

- `RedisTemplate` 는 `framework/cache/remote/CacheService` 를 통해서만 사용 — 직접 주입 금지
- Kotlin data class 저장 시 `KotlinSupportingTypeResolver` 필수 — `activateDefaultTyping` 단독 사용 금지
- 직렬화는 `GenericJackson2JsonRedisSerializer` — `@class` 타입 정보 포함
- SSL 은 운영 환경에서 활성화 권장
