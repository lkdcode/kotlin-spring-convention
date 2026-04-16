# domain/service — Domain Service

특정 Entity/Aggregate 에 속하기 어색한 비즈니스 로직.
여러 Aggregate 에 걸치거나, 외부 의존 없이 순수하게 도메인 연산만 수행.

```kotlin
// 두 Aggregate 에 걸친 비즈니스 로직
class TransferService {

    fun transfer(
        source: Account,
        target: Account,
        amount: Money,
    ) {
        require(source.id != target.id) { "동일 계좌 이체 불가" }
        source.withdraw(amount)
        target.deposit(amount)
    }
}

// 외부 데이터 없이 순수 도메인 계산
class DiscountService {

    fun calculate(
        order: Order,
        policy: DiscountPolicy,
    ): Money =
        when (policy) {
            DiscountPolicy.NONE -> Money.ZERO
            DiscountPolicy.RATE_10 -> order.totalPrice() * BigDecimal("0.1")
            DiscountPolicy.FIXED_1000 -> Money(BigDecimal("1000"))
        }
}
```

---

## Domain Service vs Application Service — 경계

둘 다 "Service" 이지만 역할이 완전히 다름.

| | Domain Service | Application Service |
|---|---|---|
| 위치 | `domain/service/` | `application/services/command/` or `query/` |
| Spring 빈 | ❌ `@Service` 없음 | ✅ `@Service` |
| 트랜잭션 | ❌ | ✅ `@Transactional` |
| 인프라 의존 | ❌ Port, Repository 주입 불가 | ✅ Port 통해 간접 의존 |
| 입력 | Domain 객체 (Entity, VO) | Command/Query Model (DTO) |
| 역할 | 도메인 규칙 실행 | UseCase 흐름 조율 + 트랜잭션 관리 |
| 예시 | `TransferService` | `TransferMoneyCommandService` |

---

## 어디에 로직을 넣을지 판단 기준

```
외부 의존(DB, 포트)이 필요한가?
    YES → Application Service
    NO  →
        단일 Entity/Aggregate 의 동작인가?
            YES → Entity/Aggregate 메서드
            NO  → Domain Service (여러 Aggregate 에 걸친 순수 도메인 연산)
```

**흔한 실수:**

```kotlin
// ❌ Domain Service 에 Port 주입 — Application Service 역할 침범
class TransferService(
    private val accountRepository: AccountRepository, // 금지
) {
    fun transfer(...) { ... }
}

// ✅ Application Service 에서 Domain Service 호출
@Service
@Transactional
class TransferMoneyCommandService(
    private val accountCommandPort: AccountCommandPort,
    private val accountQueryPort: AccountQueryPort,
    private val transferService: TransferService,       // Domain Service 주입
) : TransferMoneyUsecase {

    override fun execute(model: TransferMoneyModel) {
        val source = accountQueryPort.getById(model.sourceId)
        val target = accountQueryPort.getById(model.targetId)

        transferService.transfer(source, target, model.amount)  // 도메인 규칙 실행

        accountCommandPort.update(source)
        accountCommandPort.update(target)
    }
}
```

---

## 규칙

- `@Service` 어노테이션 없음 — Spring 빈 아님, DI 컨테이너 등록 불가
- 상태(필드) 없음 — 순수 함수 집합
- Port, Repository 직접 주입 금지 — 도메인 계층 순수성 유지
- 단일 책임 — 하나의 도메인 연산에 집중
- 이름이 어색하면 Entity/Aggregate 메서드로 이동 고려
