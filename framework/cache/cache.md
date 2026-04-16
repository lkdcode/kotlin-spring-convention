# framework/cache — 캐시

## Local vs Remote

| | Local Cache | Remote Cache |
|---|---|---|
| 저장 위치 | JVM 메모리 | Redis 서버 |
| 속도 | 매우 빠름 (ns) | 빠름 (ms) |
| 인스턴스 공유 | ❌ 인스턴스별 독립 | ✅ 모든 인스턴스 공유 |
| 서버 재시작 | 캐시 소멸 | 유지 |
| 적합한 경우 | 자주 읽고 거의 안 바뀌는 데이터 | 세션, 분산 락, 공유 캐시 |

## 디렉토리 구조

```
framework/cache/
├── cache.md          ← 전체 개요 (이 파일)
├── local/local.md    ← Caffeine (JVM In-memory)
└── remote/remote.md  ← Redis (Spring Data Redis)
```

## CacheService 인터페이스 (Remote 전용)

```kotlin
interface CacheService {
    fun getValue(key: String): Any?
    fun save(key: String, value: Any, ttl: Duration = Duration.ofMinutes(60))
    fun delete(key: String)
    fun deleteByPrefix(prefix: String): Long
    fun increment(key: String, ttl: Duration): Long
}
```

## 규칙

- Local 캐시는 Spring `@Cacheable` 어노테이션 활용
- Remote 캐시는 `CacheService` 인터페이스를 통해서만 접근 — `RedisTemplate` 직접 사용 금지
- Redis 설정은 `infra/redis/` 참조
