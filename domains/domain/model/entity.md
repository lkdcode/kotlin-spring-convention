# domain/model/entity — Entity

식별자(ID) 로 동등성을 판단하는 도메인 객체. 상태 변경 가능.

```kotlin
// 단일 Entity 예시 (Aggregate Root 와 별개 — Aggregate 는 aggregate.md 참조)
class Order(
    val id: OrderId,
    val customerId: CustomerId,
    private var status: OrderStatus,
    private val items: MutableList<OrderItem> = mutableListOf(),
) {
    // 상태 변경은 반드시 메서드를 통해서만
    fun addItem(item: OrderItem) {
        check(status == OrderStatus.DRAFT) { "확정된 주문에 아이템 추가 불가" }
        items.add(item)
    }

    fun confirm() {
        check(items.isNotEmpty()) { "아이템 없이 주문 확정 불가" }
        status = OrderStatus.CONFIRMED
    }

    fun getItems(): List<OrderItem> = items.toList()  // 방어적 복사

    // 동등성: ID 기준
    override fun equals(other: Any?) = other is Order && id == other.id
    override fun hashCode() = id.hashCode()
}
```

## 규칙
- 상태 변경은 Entity 내부 메서드로만 — setter 직접 노출 금지
- 컬렉션 반환 시 방어적 복사 (`toList()`)
- 동등성 판단: `id` 기준 (`equals` / `hashCode` 오버라이드)
- 불변식(invariant) 검증은 메서드 내부에서 (`check`, `require`)
- 생성 시 유효성 검사는 `init` 블록 또는 `companion object factory` 사용
- Aggregate 내부 Entity — Root 가 생성하므로 별도 factory 불필요
- 독립 Entity — `private constructor` + `companion object create()` factory 패턴 적용 (Aggregate Root 와 동일)
