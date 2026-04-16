# rules/test — 테스트 공통 규칙

---

## 테스트 스택

| 항목 | 기술 |
|---|---|
| 테스트 프레임워크 | Kotest (DescribeSpec, BehaviorSpec) |
| Assertion | Kotest Matchers (`shouldBe`, `shouldNotBe`) |
| Mocking | MockK (`mockk`, `every`, `verify`) |
| 테스트 데이터 | FixtureMonkey + Factory 패턴 |
| 통합 테스트 | Testcontainers (PostgreSQL, MongoDB, Redis) |
| 빌드 의존성 | `TestLibs.kt` (buildSrc) |

---

## 테스트 메서드 네이밍

**한글 백틱 BDD** — `describe` → `context` → `it`

```kotlin
class FooCommandServiceTest : DescribeSpec({

    describe("`execute` 메서드는") {
        context("유효한 CreateFooModel 이 전달되면") {
            it("Foo 를 생성할 것이다") {
                // ...
            }
        }

        context("중복된 코드가 존재하면") {
            it("ApiException 을 던질 것이다") {
                // ...
            }
        }
    }
})
```

---

## 테스트 데이터 — FixtureMonkey + Factory

### FixtureMonkeyConfig

```kotlin
@TestConfiguration
class FixtureMonkeyConfig {

    companion object {
        private val KOTLIN_FIXTURE_MONKEY: FixtureMonkey = FixtureMonkey.builder()
            .plugin(KotlinPlugin())
            .build()

        fun <T> giveMeOne(type: Class<T>): T = KOTLIN_FIXTURE_MONKEY.giveMeOne(type)
    }
}
```

### Factory 패턴

모든 테스트 대상 클래스에 `*Factory.kt` 를 작성.

```kotlin
class CreateFooModelFactory {

    companion object {
        fun create(
            name: String? = null,
            code: String? = null,
            description: String? = null,
        ): CreateFooModel {
            val base = FixtureMonkeyConfig.giveMeOne(CreateFooModel::class.java)

            return base.copy(
                name = name ?: base.name,
                code = code ?: base.code,
                description = description ?: base.description,
            )
        }
    }
}
```

### Factory 위치

```
src/test/kotlin/.../domains/foo/
├── domain/model/factory/
│   └── CreateFooModelFactory.kt       ← Domain Model Factory
├── adapter/out/infrastructure/jpa/entity/factory/
│   └── FooJpaEntityFactory.kt         ← Entity Factory
```

### Factory 규칙

- `companion object` 에 정적 팩토리 메서드
- 모든 파라미터 nullable + 기본값 null
- `FixtureMonkeyConfig.giveMeOne()` 으로 랜덤 베이스 생성
- `data class` → `.copy()`, `class` → 생성자 직접 호출
- 특수 케이스는 별도 팩토리 메서드: `createDeleted()`, `createGuest()` 등

---

## Mock 패턴

```kotlin
// 생성
val mockCommandPort = mockk<FooCommandPort>()
val mockValidator = mockk<FooValidator>()

// 설정
every { mockCommandPort.create(any()) } returns 1L
every { mockValidator.validateDuplicate(any()) } just Runs

// 검증
verify(exactly = 1) { mockCommandPort.create(any()) }
verify(exactly = 0) { mockValidator.validateDuplicate(any()) }
```

### Mock 정리

```kotlin
class FooServiceTest : DescribeSpec({

    val mockPort = mockk<FooCommandPort>()

    beforeTest { clearMocks(mockPort) }

    // ...
})
```

---

## Assertion

Kotest Matchers 사용 — JUnit Assertions, AssertJ 사용 금지.

```kotlin
result shouldBe expected
result shouldNotBe null
result.name shouldBe "foo"
list.size shouldBe 3
```

### 예외 검증

```kotlin
val exception = shouldThrow<ApiException> {
    service.execute(model)
}
exception.getApiResponseCode() shouldBe ApiResponseCode.FOO_NOT_FOUND
```

---

## 테스트 분류

| 계층 | 문서 | Spring Context | DB |
|---|---|---|---|
| Domain | `domains/domain/test/test.md` | X | X |
| Application | `domains/application/test/test.md` | X | X |
| Adapter 단위 | `domains/adapter/test/test.md` | X | X (MockK) |
| Adapter 통합 | `domains/adapter/test/test.md` | O | O (Testcontainers) |
| REST API | `domains/adapter/test/test.md` | O (슬라이스) | X |

---

## 규칙 요약

| 항목 | 규칙 |
|---|---|
| 프레임워크 | 단위: Kotest (DescribeSpec, BehaviorSpec), 통합: JUnit 5 |
| Assertion | Kotest Matchers 만 사용 |
| Mocking | MockK |
| 네이밍 | 한글 백틱 BDD |
| 테스트 데이터 | FixtureMonkey + Factory 패턴 |
| Mock 정리 | `beforeTest { clearMocks() }` |
| 통합 테스트 | Testcontainers (H2 사용 금지) |
