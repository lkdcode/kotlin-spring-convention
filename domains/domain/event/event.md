# domain/event — Domain Event

도메인에서 발생한 사실(fact). 과거형 네이밍. 불변.

```kotlin
// 마커 인터페이스
interface DomainEvent {
    val occurredAt: LocalDateTime
}

// 구체 이벤트
data class OrderConfirmedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    override val occurredAt: LocalDateTime = LocalDateTime.now(),
) : DomainEvent

data class OrderCancelledEvent(
    val orderId: OrderId,
    val reason: String,
    override val occurredAt: LocalDateTime = LocalDateTime.now(),
) : DomainEvent
```

## Application 계층에서 발행

```kotlin
// Service 에서 이벤트 발행
@Service
@Transactional
class ConfirmOrderService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: DomainEventPublisher,
) : ConfirmOrderUsecase {

    override fun confirm(orderId: OrderId) {
        val order = orderRepository.getById(orderId)
        order.confirm()                          // 이벤트 등록
        orderRepository.save(order)
        order.pullEvents().forEach(eventPublisher::publish)  // 발행
    }
}
```

## 규칙
- 이벤트명은 **과거형** (`OrderConfirmed`, `UserRegistered`)
- `data class` + `val` 필드만 — 불변
- `occurredAt` 은 발생 시각 (기본값: `LocalDateTime.now()`)
- Aggregate 내부에서 **등록**, Application 계층에서 **발행**
- 이벤트는 도메인 사실을 표현 — 처리 방법은 Handler 에서 결정
