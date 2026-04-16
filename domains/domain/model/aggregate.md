# domain/model/aggregate — Aggregate Root

트랜잭션 일관성 경계. 외부는 반드시 Root 를 통해서만 내부 접근.

```kotlin
class Order private constructor(  // factory 강제
    val id: OrderId,
    val customerId: CustomerId,
    private var status: OrderStatus,
    private val items: MutableList<OrderItem> = mutableListOf(),
    private val events: MutableList<DomainEvent> = mutableListOf(),
) {
    companion object {
        // 생성 로직을 factory 로 캡슐화
        fun create(customerId: CustomerId): Order {
            val order = Order(
                id = OrderId.generate(),
                customerId = customerId,
                status = OrderStatus.DRAFT,
            )
            order.registerEvent(OrderCreatedEvent(order.id, customerId))
            return order
        }
    }

    fun addItem(productId: ProductId, quantity: Int, price: Money) {
        check(status == OrderStatus.DRAFT) { "DRAFT 상태에서만 아이템 추가 가능" }
        items.add(OrderItem(productId, quantity, price))
    }

    fun confirm() {
        check(items.isNotEmpty()) { "아이템 없이 확정 불가" }
        status = OrderStatus.CONFIRMED
        registerEvent(OrderConfirmedEvent(id))
    }

    fun totalPrice(): Money = items.fold(Money(BigDecimal.ZERO)) { acc, item -> acc + item.totalPrice() }

    // 이벤트 수집 — Application 계층에서 발행
    fun pullEvents(): List<DomainEvent> = events.toList().also { events.clear() }
    private fun registerEvent(event: DomainEvent) = events.add(event)

    // 외부 접근 차단 — 방어적 복사
    fun getItems(): List<OrderItem> = items.toList()

    // 동등성: ID 기준 (Entity 와 동일)
    override fun equals(other: Any?) = other is Order && id == other.id
    override fun hashCode() = id.hashCode()
}
```

## Aggregate 설계 원칙
- **작게 유지** — Aggregate 경계는 트랜잭션 경계와 동일, 크면 성능 문제
- **ID 로만 참조** — 다른 Aggregate 는 객체 참조 대신 ID 참조
- **factory 메서드** — `private constructor` + `companion object create()`
- **도메인 이벤트 수집** — `pullEvents()` 로 Application 계층에 위임
- **불변식 보호** — 모든 상태 변경 시 불변식 검증
- **동등성** — `equals`/`hashCode` ID 기준 오버라이드 (Entity 와 동일)

## 단일 Entity vs Aggregate 구분
| | 단일 Entity | Aggregate |
|---|---|---|
| 내부 객체 | 없음 | Entity + VO 포함 |
| 이벤트 | 선택 | 권장 |
| 트랜잭션 경계 | 자기 자신 | Root 기준 |
