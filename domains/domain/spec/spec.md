# domain/spec — Specification

비즈니스 규칙을 객체로 표현. 조합 가능.

```kotlin
// 마커 인터페이스
interface Specification<T> {
    fun isSatisfiedBy(candidate: T): Boolean

    // 조합 연산
    infix fun and(other: Specification<T>) = AndSpecification(this, other)
    infix fun or(other: Specification<T>) = OrSpecification(this, other)
    fun not() = NotSpecification(this)
}

// 구체 Specification
class ActiveOrderSpec : Specification<Order> {
    override fun isSatisfiedBy(candidate: Order) =
        candidate.status == OrderStatus.CONFIRMED
}

class MinimumAmountSpec(private val minimum: Money) : Specification<Order> {
    override fun isSatisfiedBy(candidate: Order) =
        candidate.totalPrice() >= minimum
}

// 조합
val shippableOrder = ActiveOrderSpec() and MinimumAmountSpec(Money(BigDecimal("10000")))
val isShippable = shippableOrder.isSatisfiedBy(order)
```

## 간단한 규칙 — object 로 선언

```kotlin
// 조합이 필요 없는 단순 규칙
object OrderDeletableSpec {
    fun isSatisfiedBy(status: OrderStatus) =
        status == OrderStatus.DRAFT
}
```

## enum 기반 조건 판단

```kotlin
// enum 을 사용한 상태/조건 판단 — Spec 에서 관리
enum class DiscountType { NONE, RATE_10, FIXED_1000 }

object DiscountApplicableSpec {
    fun isSatisfiedBy(type: DiscountType, order: Order): Boolean =
        when (type) {
            DiscountType.NONE -> false
            DiscountType.RATE_10 -> order.totalPrice() >= Money(BigDecimal("10000"))
            DiscountType.FIXED_1000 -> order.totalPrice() >= Money(BigDecimal("5000"))
        }
}
```

> enum 의 `when` 분기로 **조건 판단(Boolean)** → Spec, **값 계산/변환** → Policy

## 규칙
- 조합 필요 시 `Specification<T>` 인터페이스 구현
- 단순 규칙은 `object` 로 선언
- enum 기반 조건 판단은 Spec 에서 관리
- 외부 의존성 없음 (Repository, Spring 빈 X)
- 규칙 변경 시 Spec 만 수정 — 사용처 변경 불필요
- **String, 계산값 등 Boolean 외 반환은 Spec 이 아님** — Policy 또는 유틸로 분리
