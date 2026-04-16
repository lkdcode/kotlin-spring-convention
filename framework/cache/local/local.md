# framework/cache/local — Local Cache (Caffeine)

JVM 메모리 내 캐시. 캐시별로 TTL/Size 를 명시적으로 등록.

## 의존성

```gradle
implementation("com.github.ben-manes.caffeine:caffeine")
implementation("org.springframework.boot:spring-boot-starter-cache")
```

---

## CaffeineCacheConfig

캐시를 개별 등록. 각 캐시의 TTL/Size 는 도메인의 `*CacheSpec` 에서 정의.

```kotlin
@Configuration
class CaffeineCacheConfig(
    private val jwtProperties: JwtProperties,
) {
    @Bean
    fun cacheManager(): CacheManager {
        val manager = CaffeineCacheManager()

        addCacheJwtBlackList(manager)
        // 도메인별 캐시 추가
        // addCacheFoo(manager)

        return manager
    }

    private fun addCacheJwtBlackList(manager: CaffeineCacheManager) {
        manager.registerCustomCache(
            JwtValues.BLACK_LIST_KEY,
            Caffeine.newBuilder()
                .maximumSize(10_000)
                .expireAfterWrite(jwtProperties.expired, TimeUnit.SECONDS)
                .recordStats()
                .build()
        )
    }
}
```

---

## CacheSpec 패턴

캐시 이름, TTL, Size 상수를 도메인/기능 레벨에서 정의.

```kotlin
// domains/{domain}/... 또는 application/ports/input/ 에 위치
object FooCacheSpec {
    const val CACHE_VALUE = "foo"
    const val MAXIMUM_SIZE = 1_000L
    const val CACHE_EXPIRED = 30L
    val TIME_UNIT: TimeUnit = TimeUnit.MINUTES
}

// CaffeineCacheConfig 에 등록
private fun addCacheFoo(manager: CaffeineCacheManager) {
    manager.registerCustomCache(
        FooCacheSpec.CACHE_VALUE,
        Caffeine.newBuilder()
            .maximumSize(FooCacheSpec.MAXIMUM_SIZE)
            .expireAfterWrite(FooCacheSpec.CACHE_EXPIRED, FooCacheSpec.TIME_UNIT)
            .build()
    )
}
```

---

## 사용 예시

```kotlin
// @Cacheable — 캐시 히트 시 메서드 실행 생략
@Cacheable(value = [FooCacheSpec.CACHE_VALUE], key = "#id")
fun getById(id: Long): FooDTO = queryPort.getById(FooId(id))

// @CacheEvict — 캐시 무효화
@CacheEvict(value = [FooCacheSpec.CACHE_VALUE], key = "#id")
fun update(id: Long, model: UpdateFooModel) { ... }

// CacheManager 직접 사용 — JWT 블랙리스트 등 직접 put/get
val cache = cacheManager.getCache(JwtValues.BLACK_LIST_KEY)
cache?.put(token, token)
cache?.get(token)
```

---

## SimpleCacheKeyGenerator

메서드명 + 파라미터 조합으로 키 자동 생성.

```kotlin
@Component("simpleCacheKeyGenerator")
class SimpleCacheKeyGenerator : KeyGenerator {
    override fun generate(target: Any, method: Method, vararg params: Any?): Any {
        val className = target.javaClass.simpleName
        val methodName = method.name
        val argsKey = params.joinToString(":") { it?.toString() ?: "null" }
        return "$className::$methodName::$argsKey"
    }
}
```

---

## 규칙

- 캐시는 반드시 `registerCustomCache` 로 개별 등록 — 기본 캐시 설정 사용 금지
- TTL/Size 상수는 `*CacheSpec` 으로 도메인/기능 레벨에서 정의
- JWT 블랙리스트는 `JwtProperties.expired` 와 동일 TTL 로 등록
- `recordStats()` 권장 — 캐시 히트율 모니터링
- 서버 재시작 시 캐시 소멸 — 공유/영속 캐시 필요 시 `remote/` 사용
