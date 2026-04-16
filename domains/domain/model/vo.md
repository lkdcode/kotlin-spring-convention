# domain/model/vo — Value Object

식별자 없음. 값으로 동등성 판단. 불변.

```kotlin
// 단순 래핑 — @JvmInline 으로 런타임 오버헤드 없음
@JvmInline
value class Money(val amount: BigDecimal) {
    init {
        require(amount >= BigDecimal.ZERO) { "금액은 0 이상이어야 합니다" }
    }

    operator fun plus(other: Money) = Money(amount + other.amount)
    operator fun minus(other: Money) = Money(amount - other.amount)
    operator fun compareTo(other: Money) = amount.compareTo(other.amount)
}

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "유효하지 않은 이메일: $value" }
    }
}

// 여러 필드 — data class 사용
data class Address(
    val street: String,
    val city: String,
    val zipCode: String,
) {
    init {
        require(zipCode.length == 5) { "우편번호는 5자리여야 합니다" }
    }
}
```

## 일급 컬렉션 (First-Class Collection)

도메인 객체 컬렉션을 감쌀 때 사용. `List<도메인객체>` 를 직접 노출하지 않고 래핑.

```kotlin
// ❌ 원시 컬렉션 직접 노출
fun createAll(items: List<OrderItem>)

// ✅ 일급 컬렉션으로 래핑
data class OrderItems(val items: List<OrderItem>) {
    init {
        require(items.isNotEmpty()) { "주문 항목은 최소 1개 이상이어야 합니다" }
    }
    fun totalPrice(): Money = items.fold(Money.ZERO) { acc, item -> acc + item.price }
}

fun createAll(items: OrderItems)
```

## 규칙
- 단일 값 래핑: `@JvmInline value class` — 런타임에 원시타입으로 최적화
- 복합 값: `data class` — `equals` / `hashCode` / `copy` 자동 제공
- 모든 필드 `val` — 불변 필수
- `init` 블록에서 유효성 검사 (`require`)
- 비즈니스 연산 메서드 포함 가능 (`plus`, `compareTo`, `totalPrice` 등)
- setter, 상태 변경 메서드 금지
- `List<도메인객체>` 직접 파라미터 노출 금지 — 일급 컬렉션으로 래핑
