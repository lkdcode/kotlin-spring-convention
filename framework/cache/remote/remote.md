# framework/cache/remote — Remote Cache (Redis)

Spring Data Redis 기반. 모든 인스턴스가 공유하는 분산 캐시.
Redis 서버 설정은 `infra/redis/` 참조.

## 의존성

```gradle
implementation("org.springframework.boot:spring-boot-starter-data-redis")
```

## CacheService 구현체

```kotlin
@Service
class RedisCacheService(
    private val redisTemplate: RedisTemplate<String, Any>,
) : CacheService {

    override fun getValue(key: String): Any? =
        redisTemplate.opsForValue().get(key)

    override fun save(
        key: String,
        value: Any,
        ttl: Duration,
    ) {
        redisTemplate.opsForValue().set(key, value, ttl)
    }

    override fun delete(key: String) {
        redisTemplate.delete(key)
    }

    override fun deleteByPrefix(prefix: String): Long {
        val keys = redisTemplate.keys("$prefix*")
        return if (keys.isNullOrEmpty()) 0L
        else redisTemplate.unlink(keys) ?: 0L
    }

    override fun increment(
        key: String,
        ttl: Duration,
    ): Long {
        val count = redisTemplate.opsForValue().increment(key) ?: 0L
        if (count == 1L) redisTemplate.expire(key, ttl)
        return count
    }
}
```

## 사용 예시

```kotlin
@Service
class FooQueryService(
    private val queryPort: FooQueryPort,
    private val cache: CacheService,
) : FetchFooUsecase {

    override fun getById(id: FooId): FetchFooDTO {
        val key = "foo:${id.value}"

        @Suppress("UNCHECKED_CAST")
        return cache.getValue(key) as? FetchFooDTO
            ?: queryPort.getById(id).also {
                cache.save(key, it, Duration.ofMinutes(30))
            }
    }
}
```

## 캐시 키 컨벤션

```
{도메인}:{식별자}
foo:123
user:456
session:token-value
rate-limit:userId:endpoint
```

## 주요 활용 패턴

| 패턴 | 예시 | TTL |
|---|---|---|
| 조회 캐시 | `foo:123` | 데이터 변경 빈도에 따라 |
| 세션 | `session:{token}` | 로그인 만료 시간 |
| Rate Limiting | `rate:{userId}:{endpoint}` | 1분 ~ 1시간 |
| 분산 락 | `lock:{resource}` | 작업 최대 시간 |
| 블랙리스트 | `BLACK_LIST:{token}` | JWT 만료 시간 |

## 규칙

- `CacheService` 인터페이스만 주입 — `RedisTemplate` 직접 사용 금지
- 키는 `{도메인}:{식별자}` 패턴 준수
- `deleteByPrefix` 는 대용량 키 삭제 시 성능 주의 — 운영 환경에서는 SCAN 기반 비동기 삭제 고려
- TTL 없는 캐시 저장 금지 — 반드시 만료 시간 설정
