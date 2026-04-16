# domain/test — Domain 단위 테스트

Spring Context 없음. Mock 없음. 순수 로직 검증.

---

## 대상

| 대상 | 테스트 내용 |
|---|---|
| Model (Command/Query) | `init` 블록 `require()` 검증, 계산 프로퍼티 |
| Value Object | 생성 검증, 동등성 |
| Entity | 상태 변경 메서드, 불변식 검증, equals/hashCode |
| Spec | Boolean 반환값, 조합 (and/or/not) |
| Policy | 계산 결과 |
| Aggregate | factory 메서드, 이벤트 발행, 불변식 |
| Domain Service | 여러 Aggregate 에 걸친 순수 도메인 연산 |

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

## Aggregate 테스트

```kotlin
class OrderTest : DescribeSpec({

    describe("`Order.create` 는") {
        it("DRAFT 상태로 생성하고 OrderCreatedEvent 를 등록할 것이다") {
            val order = Order.create(CustomerId(1L))

            order.getItems() shouldHaveSize 0
            val events = order.pullEvents()
            events shouldHaveSize 1
            events.first() shouldBeInstanceOf OrderCreatedEvent::class
        }
    }

    describe("`confirm` 은") {
        context("아이템이 있으면") {
            it("CONFIRMED 로 전환하고 OrderConfirmedEvent 를 등록할 것이다") {
                val order = Order.create(CustomerId(1L))
                order.addItem(ProductId(1L), 2, Money(BigDecimal("5000")))
                order.pullEvents() // create 이벤트 소비

                order.confirm()

                val events = order.pullEvents()
                events shouldHaveSize 1
                events.first() shouldBeInstanceOf OrderConfirmedEvent::class
            }
        }

        context("아이템이 없으면") {
            it("IllegalStateException 을 던질 것이다") {
                val order = Order.create(CustomerId(1L))

                shouldThrow<IllegalStateException> { order.confirm() }
            }
        }
    }
})
```

## Domain Service 테스트

```kotlin
class TransferServiceTest : DescribeSpec({

    describe("`transfer` 는") {
        context("동일 계좌가 아니면") {
            it("출금과 입금이 수행될 것이다") {
                val source = Account.create(AccountId(1L), Money(BigDecimal("10000")))
                val target = Account.create(AccountId(2L), Money(BigDecimal("5000")))
                val service = TransferService(accountRepository)

                service.transfer(source.id, target.id, Money(BigDecimal("3000")))

                source.balance shouldBe Money(BigDecimal("7000"))
                target.balance shouldBe Money(BigDecimal("8000"))
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
