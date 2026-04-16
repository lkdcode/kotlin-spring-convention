# adapter/test — Adapter 테스트

Adapter 테스트는 **단위 테스트**와 **통합 테스트** 두 가지로 나뉜다.

---

## 단위 테스트 vs 통합 테스트

| | 단위 테스트 | 통합 테스트 |
|---|---|---|
| Spring Context | X | O |
| DB | X (MockK) | O (Testcontainers) |
| 프레임워크 | Kotest DescribeSpec | JUnit 5 + IntegrationTestSupport |
| 속도 | 빠름 | 느림 (컨테이너 기동) |
| 검증 대상 | 로직 분기, 매핑 | 실제 SQL, 쿼리 정합성 |

---

## 단위 테스트 (Kotest + MockK)

Spring Context 없이 Adapter 내부 로직만 검증. Application 테스트와 동일한 방식.

### Command Adapter

```kotlin
class FooCommandAdapterTest : DescribeSpec({

    val mockRepository = mockk<FooJpaRepository>()
    val mockMapper = mockk<FooCommandMapper>()
    val adapter = FooCommandAdapter(mockRepository, mockMapper)

    beforeTest { clearMocks(mockRepository, mockMapper) }

    describe("`save` 메서드는") {
        context("CreateFooModel 이 전달되면") {
            it("Mapper 로 변환 후 저장할 것이다") {
                // Given
                val model = CreateFooModelFactory.create()
                val entity = FooJpaEntityFactory.create(id = 1L)
                every { mockMapper.convert(any()) } returns entity
                every { mockRepository.save(any()) } returns entity

                // When
                adapter.save(model)

                // Then
                verify(exactly = 1) { mockMapper.convert(model) }
                verify(exactly = 1) { mockRepository.save(entity) }
            }
        }
    }
})
```

### JPA Entity

Spring Context 없이 Entity 메서드만 검증.

```kotlin
class FooJpaEntityTest : DescribeSpec({

    describe("`update` 메서드는") {
        it("필드를 변경할 것이다") {
            val entity = FooJpaEntityFactory.create()
            val model = UpdateFooModelFactory.create(name = "변경된 이름")

            entity.update(model)

            entity.name shouldBe "변경된 이름"
        }
    }

    describe("`delete` 메서드는") {
        it("soft delete 상태로 변경할 것이다") {
            val entity = FooJpaEntityFactory.create()

            entity.delete()

            entity.isDeleted() shouldBe true
            entity.deletedAt shouldNotBe null
        }
    }
})
```

### Validator Adapter

```kotlin
class FooValidatorAdapterTest : DescribeSpec({

    val mockDsl = mockk<DSLContext>()
    val adapter = FooValidatorAdapter(mockDsl)

    beforeTest { clearMocks(mockDsl) }

    describe("`validateDuplicate` 메서드는") {
        context("중복된 코드가 존재하면") {
            it("ApiException 을 던질 것이다") {
                // Given
                every { mockDsl.fetchExists(any()) } returns true

                // When & Then
                shouldThrow<ApiException> {
                    adapter.validateDuplicate("DUPLICATE_CODE")
                }.getApiResponseCode() shouldBe ApiResponseCode.FOO_DUPLICATE_CODE
            }
        }

        context("중복된 코드가 없으면") {
            it("정상 통과할 것이다") {
                // Given
                every { mockDsl.fetchExists(any()) } returns false

                // When & Then
                adapter.validateDuplicate("UNIQUE_CODE")
            }
        }
    }
})
```

---

## 통합 테스트 (JUnit 5 + Testcontainers)

실제 DB 로 SQL/쿼리 정합성을 검증. Kotest 가 아닌 **JUnit 5** 사용.

> Kotlin 은 다중 상속 불가 — `IntegrationTestSupport`(abstract class)와 `DescribeSpec`(class)를 동시에 상속할 수 없다.
> 통합 테스트는 JUnit 5 + `@Nested` + `@DisplayName` 으로 BDD 구조를 표현한다.

### Testcontainers 라이프사이클

```
JVM 시작 → object 싱글턴 컨테이너 기동 (1회)
         → Spring Context 로드 → Flyway 마이그레이션 (1회)
         → 테스트 실행 → 데이터 삽입 → 검증 → @AfterEach 정리 (반복)
JVM 종료 → 컨테이너 자동 파괴
```

- **스키마**: `src/main/resources/db/migration/V*.sql` — 프로덕션과 동일한 Flyway 마이그레이션 그대로 사용
- **컨테이너**: JVM 당 1회만 기동, 스키마도 1회만 셋업
- **데이터 정리**: `@AfterEach` 에서 `repository.deleteAll()` 또는 `@Transactional` 롤백

### 컨테이너 싱글턴

JVM 당 1회만 기동. `object` 로 선언.

```kotlin
object PostgresTestContainer {
    private val container: PostgreSQLContainer<*> = PostgreSQLContainer(
        DockerImageName.parse("postgres:17-alpine")
    ).apply {
        withDatabaseName("foo_test")
        withUsername("test")
        withPassword("test")
    }

    init { container.start() }

    fun getJdbcUrl(): String = container.jdbcUrl
    fun getUsername(): String = container.username
    fun getPassword(): String = container.password
}
```

```kotlin
object MongoTestContainer {
    private val container: MongoDBContainer = MongoDBContainer(
        DockerImageName.parse("mongo:7.0")
    )

    init { container.start() }

    fun getConnectionString(): String {
        val baseUri = container.connectionString
        return if (baseUri.endsWith("/")) "${baseUri}foo_test"
        else "$baseUri/foo_test"
    }
}
```

```kotlin
object RedisTestContainer {
    private const val REDIS_PORT = 6379

    private val container: GenericContainer<*> = GenericContainer(
        DockerImageName.parse("redis:7-alpine")
    ).apply { withExposedPorts(REDIS_PORT) }

    init { container.start() }

    fun getHost(): String = container.host
    fun getPort(): Int = container.getMappedPort(REDIS_PORT)
}
```

#### IntegrationTestSupport

모든 통합 테스트의 베이스 클래스.

```kotlin
@SpringBootTest
@ActiveProfiles("test")
abstract class IntegrationTestSupport {

    companion object {
        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            // PostgreSQL + Flyway
            registry.add("spring.datasource.url") { PostgresTestContainer.getJdbcUrl() }
            registry.add("spring.datasource.username") { PostgresTestContainer.getUsername() }
            registry.add("spring.datasource.password") { PostgresTestContainer.getPassword() }
            registry.add("spring.flyway.url") { PostgresTestContainer.getJdbcUrl() }
            registry.add("spring.flyway.user") { PostgresTestContainer.getUsername() }
            registry.add("spring.flyway.password") { PostgresTestContainer.getPassword() }

            // MongoDB
            registry.add("spring.data.mongodb.uri") { MongoTestContainer.getConnectionString() }

            // Redis
            registry.add("spring.data.redis.host") { RedisTestContainer.getHost() }
            registry.add("spring.data.redis.port") { RedisTestContainer.getPort() }
        }
    }
}
```

#### 디렉토리 구조

```
src/test/kotlin/.../framework/testcontainers/
├── IntegrationTestSupport.kt
├── PostgresTestContainer.kt
├── MongoTestContainer.kt
└── RedisTestContainer.kt
```

### 통합 테스트 예시

실제 Repository, DSLContext 를 Spring 에서 주입받아 실제 DB 에 쿼리.

```kotlin
@DisplayName("FooCommandAdapter 통합 테스트")
class FooCommandAdapterIntegrationTest : IntegrationTestSupport() {

    @Autowired
    private lateinit var adapter: FooCommandAdapter

    @Autowired
    private lateinit var repository: FooJpaRepository

    @AfterEach
    fun cleanup() {
        repository.deleteAll()
    }

    @Nested
    @DisplayName("`save` 메서드는")
    inner class Save {

        @Test
        @DisplayName("실제 DB 에 저장할 것이다")
        fun `실제 DB 에 저장할 것이다`() {
            // Given
            val model = CreateFooModelFactory.create(
                name = "테스트",
                code = "TEST001",
            )

            // When
            adapter.save(model)

            // Then
            val saved = repository.findAll()
            saved shouldHaveSize 1
            saved.first().name shouldBe "테스트"
            saved.first().code shouldBe "TEST001"
        }
    }

    @Nested
    @DisplayName("`update` 메서드는")
    inner class Update {

        @Test
        @DisplayName("실제 DB 의 데이터를 변경할 것이다")
        fun `실제 DB 의 데이터를 변경할 것이다`() {
            // Given
            val entity = repository.save(FooJpaEntityFactory.create())
            val model = UpdateFooModelFactory.create(name = "변경된 이름")

            // When
            adapter.update(entity.id!!, model)

            // Then
            val updated = repository.findById(entity.id!!).orElseThrow()
            updated.name shouldBe "변경된 이름"
        }
    }
}
```

---

## Input Adapter 테스트 (REST)

`@WebMvcTest` 슬라이스 테스트. MockMvc 는 자동 주입.

```kotlin
@WebMvcTest(FooCommandApi::class)
@ActiveProfiles("test")
@DisplayName("FooCommandApi")
class FooCommandApiTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockkBean
    private lateinit var createUsecase: CreateFooUsecase

    @Nested
    @DisplayName("POST `/api/foos`")
    inner class CreateFoo {

        @Test
        @DisplayName("유효한 요청이면 201 Created 를 반환할 것이다")
        fun `유효한 요청이면 201 Created 를 반환할 것이다`() {
            // Given
            every { createUsecase.execute(any()) } just Runs

            // When & Then
            mockMvc.perform(
                post("/api/foos")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content("""{"name": "foo", "code": "FOO001"}""")
            )
                .andExpect(status().isCreated)
        }
    }
}
```

> `@WebMvcTest` 는 슬라이스 테스트 — 웹 계층만 로드. `IntegrationTestSupport` (`@SpringBootTest`) 상속 금지.
> `@MockkBean` 은 `com.ninja-squad:springmockk` 의존성 필요.

---

## application-test.yml

```yaml
spring:
  profiles:
    active: test

  # Testcontainers 가 @DynamicPropertySource 로 주입
  datasource:
    url: # DynamicPropertySource
    username: # DynamicPropertySource
    password: # DynamicPropertySource

  flyway:
    enabled: true

  jpa:
    hibernate:
      ddl-auto: validate   # Flyway 로 스키마 관리, JPA 는 검증만

logging:
  level:
    root: WARN
    com.example: DEBUG
```

---

## 규칙

### 단위 테스트
- Kotest DescribeSpec + MockK
- Spring Context 사용 금지
- 로직 분기, 매핑 검증에 집중

### 통합 테스트
- JUnit 5 + `@Nested` + `@DisplayName` (한글 백틱 메서드명)
- `IntegrationTestSupport` 상속 필수
- **H2 사용 금지** — Testcontainers 로 실제 DB 사용
- 컨테이너는 `object` 싱글턴 — 테스트 전체에서 1회만 기동
- `@ActiveProfiles("test")` — `application-test.yml` 활성화
- Flyway 마이그레이션으로 스키마 관리 — `ddl-auto: validate`
- `@AfterEach` 로 테스트 데이터 정리
- 외부 서비스 (FCM, Slack, SMS 등) 는 `@MockkBean` 으로 대체
- Docker 가 필요 — CI 환경에서도 Docker 실행 가능해야 함
- MockMvc 는 `@Autowired` 자동 주입 — `mockk<MockMvc>()` 금지
