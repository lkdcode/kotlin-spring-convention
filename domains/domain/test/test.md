# domain/test — Domain 단위 테스트

Spring Context 없음. Mock 없음. 순수 로직 검증.

---

## 대상

| 대상 | 테스트 내용 |
|---|---|
| Model (Command/Query) | `init` 블록 `require()` 검증, 계산 프로퍼티 |
| Value Object | 생성 검증, 동등성 |
| Spec | Boolean 반환값, 조합 (and/or/not) |
| Policy | 계산 결과 |
| Aggregate | factory 메서드, 이벤트 발행 |

---

## Model 테스트

FixtureMonkey 로 무작위 데이터 생성 → `require()` 검증 로직 엣지 케이스 탐지.

```kotlin
class CreateFooModelTest : DescribeSpec({

    describe("`CreateFooModel` 생성은") {
        context("유효한 값이 전달되면") {
            it("정상적으로 생성될 것이다") {
                val model = CreateFooModelFactory.create()

                model.name shouldNotBe null
                model.code shouldNotBe null
            }
        }

        context("name 이 빈 문자열이면") {
            it("IllegalArgumentException 을 던질 것이다") {
                shouldThrow<IllegalArgumentException> {
                    CreateFooModelFactory.create(name = "")
                }
            }
        }
    }
})
```

## Spec 테스트

BehaviorSpec + 파라미터화 테스트.

```kotlin
class FooPasswordSpecTest : BehaviorSpec({

    given("`FooPasswordSpec.REGEX`") {
        `when`("유효한 비밀번호가 전달되면") {
            then("true 를 반환할 것이다") {
                forAll(
                    row("Abcd1234!"),
                    row("Test@9876"),
                ) { password ->
                    FooPasswordSpec.REGEX.matches(password) shouldBe true
                }
            }
        }

        `when`("유효하지 않은 비밀번호가 전달되면") {
            then("false 를 반환할 것이다") {
                forAll(
                    row("1234"),
                    row("abcdefgh"),
                    row(""),
                ) { password ->
                    FooPasswordSpec.REGEX.matches(password) shouldBe false
                }
            }
        }
    }
})
```

---

## 규칙

- Spring Context 사용 금지 — `@SpringBootTest`, `@DataJpaTest` 등 금지
- Mock 사용 금지 — 순수 로직만 테스트
- FixtureMonkey + Factory 로 테스트 데이터 생성
- `DescribeSpec` 기본, Spec/Policy 는 `BehaviorSpec` 허용
- 파라미터화 테스트: `forAll(row(...))` 사용
