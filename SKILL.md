---
name: kotlin-spring-convention
description: Use when implementing any Kotlin Spring Boot backend feature — CRUD, usecase, domain model, aggregate, entity, value object, service, adapter, port, repository, JPA entity, jOOQ query, REST controller, command service, query service, or any hexagonal architecture (Ports & Adapters) layer. Triggers on Kotlin backend code generation, refactoring, or review.
---

# Kotlin Spring Convention (Hexagonal Architecture)

## Stack
- Kotlin / JVM 21 / Spring Boot 3.x (WebMVC) / Virtual Thread
- Command: JPA (단순 저장/수정/삭제) + jOOQ DSLContext (벌크/복잡한 쓰기)
- Query: jOOQ DSLContext (복잡한 조회/페이징/JOIN) + JPA 확장함수 (단순 단건 조회)
- Virtual Thread: `synchronized` 대신 `ReentrantLock` 사용

---

## Domain Layer (`domains/domain/`)

### Aggregate Root
```kotlin
class Order private constructor(...) {          // private constructor 필수
    companion object {
        fun create(...): Order { ... }          // factory 메서드로만 생성
    }
    fun pullEvents(): List<DomainEvent> = ...   // Application 계층에서 발행
    fun getItems(): List<OrderItem> = items.toList()  // 방어적 복사
}
```
- 외부는 Root 통해서만 접근, 다른 Aggregate는 ID 참조만

### Entity
- 상태 변경은 메서드를 통해서만 (setter 노출 금지)
- `equals`/`hashCode` → ID 기준 오버라이드
- 컬렉션 반환 시 `toList()` 방어적 복사

### Value Object
```kotlin
@JvmInline value class Money(val amount: BigDecimal)  // 단일 값
data class Address(val street: String, ...)            // 복합 값
```
- 모든 필드 `val`, `init` 블록에서 `require()` 검증

### Domain Exception
```kotlin
sealed class DomainException(message: String) : RuntimeException(message)
```
- HTTP 상태코드, ApiResponse 모름 — Adapter 경계에서 변환

### Spec / Policy / Event
- **Spec**: `Boolean` 반환, `and/or/not` 조합 가능, 외부 의존 없음
- **Policy**: 계산값 반환, 상태 없으면 `object`
- **Event**: 과거형 네이밍(`OrderConfirmedEvent`), `data class` + `val`만, Aggregate에서 등록 → Application에서 발행

### Repository (조건부 도입)
단순 CRUD는 **생략** — `domains/application/output/` 포트로 충분.
DomainService가 여러 Aggregate를 직접 조회할 때만 도입.

---

## Application Layer

### Command Service (`domains/application/services/command/`)

```kotlin
@Service
@Transactional
class CreateFooCommandService(
    private val commandPort: FooCommandPort,
    private val validator: FooValidator,
) : CreateFooUsecase {

    override fun execute(model: CreateFooModel) {
        validator.validateDuplicate(model.code)
        commandPort.create(model)
    }
}
```

> 기본은 포트 직접 호출. DSL 패턴은 복잡한 흐름에서 사용자 승인 후 도입 (`domains/application/services/command/command.md` 참조)

### Query Service (`domains/application/input/`)
```kotlin
@Service
@Transactional(readOnly = true)
class FetchFooService(private val queryPort: FooQueryPort) : FetchFooUsecase
```

### Output Ports — 인터페이스 (`domains/application/output/`)

| 인터페이스 | 반환 | throw | 역할 |
|---|---|---|---|
| `*CommandPort` | — | — | save / update / delete |
| `*QueryPort` | DTO | — | fetchList / fetchLookupList |
| `*Validator` | `Unit` | O | 데이터 유효성 (중복, 존재) |
| `*Guard` | `Unit` | O | 비즈니스 규칙 (상태 전이, 삭제 조건) |
| `*Checker` | `Boolean` | X | 복잡한 분기 — 호출부가 처리 |

---

## Adapter Layer

### Input (`domains/adapter/input/`)
```kotlin
@RestController
class FooCommandApi(
    private val createUsecase: CreateFooUsecase,  // Usecase 별로 분리 주입
    private val updateUsecase: UpdateFooUsecase,
    private val deleteUsecase: DeleteFooUsecase,
) { ... }

@RestController
class FooQueryApi(private val fetchUsecase: FetchFooUsecase) // Fetch는 하나만
```
- 반환: `ApiResponse.success*()` 팩토리 메서드 사용
- QueryString: QueryApi 파일 하단에 같이 선언, `BaseQueryString` 상속

### Output — Command (`domains/adapter/output/command/`)
```kotlin
@Service
class FooCommandAdapter(
    private val repository: FooJpaRepository,  // 단순 저장/수정/삭제
    private val dsl: DSLContext,               // 벌크/복잡한 쓰기
) : FooCommandPort
```

### Output — Query (`domains/adapter/output/query/`)
- `*QueryAdapter` / `*ValidatorAdapter` / `*GuardAdapter` / `*CheckerAdapter`
- **전부 jOOQ DSLContext** — JPA Repository 의존 없음
- 테이블 상수는 `companion object` 선언

---

## JPA vs jOOQ 선택 기준

| 작업 | 사용 |
|---|---|
| 단순 save / update / delete | JPA Repository |
| 벌크 UPDATE / DELETE | jOOQ DSLContext |
| 단순 단건 조회 (존재 확인, ID 조회) | JPA Repository 확장함수 |
| 복잡한 조회, 페이징, JOIN, 조건 검색 | jOOQ DSLContext |

## JPA Entity
```kotlin
@Entity @Table(name = "foos")
class FooJpaEntity(...) : BaseEntity() {
    fun update(model: UpdateFooModel) { ... }  // 상태 변경은 메서드로
    fun delete() { ... }                        // soft delete
}

interface FooJpaRepository : JpaRepository<FooJpaEntity, Long>
fun FooJpaRepository.loadById(id: Long): FooJpaEntity =
    findById(id).orElseThrow { throw ApiException(ApiResponseCode.FOO_NOT_FOUND) }
```

---

## 의존 방향 (헥사고날)

```
Controller (Adapter-in)
  → UseCase Interface (Application-in)
      → Service impl
          → Port Interface (Application-out)
              → Adapter impl (Adapter-out)
                  → JPA Repository / jOOQ DSLContext
```

---

## Cloud (외부 프로바이더)
| `cloud/cloud.md` | 클라우드 벤더 개요 (AWS, NCP) |
| `cloud/aws/s3/s3.md` | AWS S3 — AwsS3Properties + AwsS3Service |
| `cloud/ncp/sms/sms.md` | NCP SMS — HMAC 서명 + NcpSmsSendService |
| `infra/infra.md` | 인프라 서비스 개요 (Redis, Kafka, Slack) |
| `infra/redis/redis.md` | Redis — RedisProperties, RedisConfig, LettuceConnectionFactory |
| `infra/kafka/kafka.md` | Kafka — KafkaTopic, Consumer/ProducerConfig, KafkaProducer |
| `infra/slack/slack.md` | Slack — SlackProperties, SlackService, SlackMessageCreator |
## Framework Storage
| `framework/storage/storage.md` | 3가지 방식 비교, StoragePathUtil, 선택 기준 |
| `framework/storage/port/port.md` | DirectStoragePort / PresignedStoragePort 인터페이스 |
| `framework/storage/local/direct.md` | Local Direct — multipart 직접 업로드 |
| `framework/storage/local/presigned.md` | Local Pre-signed — 로컬 URL 발급 |
| `framework/storage/s3/s3.md` | AWS S3 Pre-signed URL |

## Framework Cache
| `framework/cache/cache.md` | Local vs Remote 비교, CacheService 인터페이스 |
| `framework/cache/local/local.md` | Caffeine — @Cacheable, SimpleCacheKeyGenerator |
| `framework/cache/remote/remote.md` | Redis — RedisCacheService, 키 컨벤션, 활용 패턴 |

## Framework JPA
| `framework/jpa/jpa.md` | BaseTimeEntity / BaseEntity / Validatable / validate() |

## Framework 유틸리티
| `framework/util/util.md` | TimeUtil / throwIf / throwUnless / isNot / isTrue / isFalse / ifNull |

## API 프레임워크
| `framework/api/response/response.md` | ApiResponse factory 메서드 + ApiResponseCode prefix 규칙 |
| `framework/api/exception/exception.md` | ApiException — ApiResponseCode 기반 커스텀 예외 |
| `framework/api/advice/advice.md` | ApiAdvice — 예외 처리 우선순위, 로그 레벨 기준 |
| `framework/api/client/client.md` | ApiWebClient — 외부 API 호출 래퍼 |

## Security (횡단 관심사)
| `framework/security/security.md` | 전체 인증 흐름, 필터 체인 개요 |
| `framework/security/jwt/jwt.md` | JwtCreator/Parser/Validator/Remover/Service |
| `framework/security/filter/filter.md` | JwtAuthenticationFilter, JwtLoginFilter, ApiSecurityFilter |
| `framework/security/auth/auth.md` | AuthenticationDTO, UserDetailsServiceAdapter, AuditorAware |
| `framework/security/handler/handler.md` | 401/403 핸들러, CORS 설정 |
| `framework/security/aop/aop.md` | @CheckRole AOP 권한 제어 |

## 공통 규칙 (계층 무관)
- `rules/db.md` — 테이블/컬럼/인덱스 네이밍 컨벤션, 도메인 우선 테이블 구조
- `rules/mongodb.md` — MongoDB 컬렉션/필드/인덱스 네이밍 컨벤션
- `rules/gitignore.md` — .gitignore 템플릿, 민감 정보 커밋 금지 항목
- `rules/build.md` — buildSrc 의존성 관리, *Libs.kt 패턴, gradle/config 주입 방식
- `rules/resources.md` — resources 디렉토리 구조, 환경별 설정 분리, Flyway 파일 네이밍
- `rules/code-style.md` — 파라미터 개행, trailing comma, expression body, 어노테이션 순서 등
- `rules/dependency.md` — 구현체 아닌 인터페이스 의존, adapter 간 직접 의존 금지
- `rules/change-scope.md` — 커밋 범위 규칙, fix/refactor/feat 분리, 부수효과 점검
- `rules/test.md` — 테스트 공통 규칙 (Kotest, MockK, FixtureMonkey, Testcontainers)

## Full Reference (상세 규칙)

Read the relevant files from the plugin root (`kotlin-spring-convention/`):

| 경로 | 내용 |
|---|---|
| `stack.md` | 기술 스택, Virtual Thread, JPA vs jOOQ |
| `domains/domain/model/aggregate.md` | Aggregate Root 설계 원칙 |
| `domains/domain/model/entity.md` | Entity 규칙 |
| `domains/domain/model/vo.md` | Value Object 규칙 |
| `domains/domain/event/event.md` | 도메인 이벤트 |
| `domains/domain/exception/exception.md` | 도메인 예외 |
| `domains/domain/policy/policy.md` | Policy 패턴 |
| `domains/domain/spec/spec.md` | Specification 패턴 |
| `domains/domain/repository/repository.md` | Repository 도입 기준 |
| `domains/domain/service/service.md` | Domain Service |
| `domains/application/ports/input/command/command.md` | Command Usecase 인터페이스 (inbound port) |
| `domains/application/ports/input/command/model/model.md` | Command Model (CreateXxxModel, UpdateXxxModel) |
| `domains/application/ports/input/query/query.md` | Query Usecase 인터페이스 (inbound port) |
| `domains/application/ports/input/query/dto/dto.md` | Query DTO (FetchXxxDTO, FetchXxxDTOList, XxxLookupDTO) |
| `domains/application/ports/input/query/common/common.md` | Query 공통 규약 (Pagination, ParamConditionList, FooConditionKey) |
| `domains/application/services/command/command.md` | *CommandService 구현체 — 포트 직접 호출 |
| `domains/application/services/query/query.md` | *QueryService 구현체 |
| `domains/application/ports/output/command/command.md` | CommandPort |
| `domains/application/ports/output/query/query.md` | QueryPort — 단건 `getXxx`, 다건 `fetchXxx`, 단건 null 반환 금지 |
| `domains/application/ports/output/validator/validator.md` | Validator — `validateXxx`, Unit, 무조건 throw |
| `domains/application/ports/output/guard/guard.md` | Guard — `requireXxx`, Unit, 무조건 throw |
| `domains/application/ports/output/checker/checker.md` | Checker — Boolean 반환, throw 금지 |
| `domains/application/ports/output/publisher/publisher.md` | Publisher (이벤트 발행 포트) |
| `domains/adapter/input/rest/common/common.md` | ApiResponse typealias + ApiBody + factory functions |
| `domains/adapter/input/rest/common/dto.md` | Request / Response DTO + Mapper 규칙 (command/query 공통) |
| `domains/adapter/input/rest/command/command.md` | REST CommandApi |
| `domains/adapter/input/rest/query/query.md` | REST QueryApi + QueryString |
| `domains/adapter/input/rest/common/swagger.md` | Swagger 문서 작성 양식 |
| `domains/adapter/input/web/command/command.md` | Web Command — *FormController (SSR, PRG 패턴) |
| `domains/adapter/input/web/query/query.md` | Web Query — *PageController (SSR, Model + 템플릿) |
| `domains/adapter/input/event/spring/spring.md` | Spring 이벤트 수신 (@TransactionalEventListener) |
| `domains/adapter/input/event/kafka/kafka.md` | Kafka 이벤트 수신 (@KafkaListener) |
| `domains/adapter/input/schedule/schedule.md` | Quartz Job + Config + Cron(@Profile) 구조 |
| `domains/application/ports/input/schedule/schedule.md` | Schedule 인바운드 포트 (*ScheduleInput) |
| `domains/adapter/output/command/command.md` | CommandAdapter (JPA + jOOQ) + CommandMapper |
| `domains/adapter/output/query/query.md` | QueryAdapter (jOOQ) |
| `domains/adapter/output/checker/checker.md` | CheckerAdapter (Boolean, no throw) |
| `domains/adapter/output/validator/validator.md` | ValidatorAdapter + GuardAdapter (Unit, throws) |
| `domains/adapter/output/publisher/publisher.md` | 브로드캐스트 어댑터 — socket/*PublisherAdapter, spring/*PublisherAdapter |
| `domains/adapter/output/producer/producer.md` | 이벤트 발행 어댑터 — kafka/*ProducerAdapter |
| `domains/adapter/output/notifier/notifier.md` | 알림 어댑터 — fcm/*NotifierAdapter |
| `domains/application/ports/output/publisher/publisher.md` | 브로드캐스트 포트 (WebSocket, Spring Event) |
| `domains/application/ports/output/notifier/notifier.md` | 알림 포트 (FCM, APNS, SMS) |
| `domains/application/ports/output/producer/producer.md` | 이벤트 발행 포트 (Kafka, RabbitMQ) |
| `domains/adapter/output/infrastructure/jpa/entity/entity.md` | JPA / R2DBC Entity + BaseEntity (var/val, @Column 제약, Instant, validate()) |
| `domains/adapter/output/infrastructure/jpa/repository/repository.md` | JPA Repository + 확장함수 패턴 (@Query native, loadById) |
| `domains/adapter/output/infrastructure/mongo/doc/doc.md` | MongoDB Document (도메인 설계 vs 외부 연동) |
| `domains/adapter/output/infrastructure/mongo/repository/repository.md` | MongoDB Repository (ReactiveMongoRepository, MongoRepository) |
| `infra/mongodb/mongodb.md` | MongoDB 연결 설정, 인덱스 관리 |
| `domains/domain/test/test.md` | Domain 단위 테스트 (순수 로직, Mock 없음, FixtureMonkey) |
| `domains/application/test/test.md` | Application Service 테스트 (MockK, Port mocking) |
| `domains/adapter/test/test.md` | Adapter 통합 테스트 (Testcontainers, IntegrationTestSupport) |
