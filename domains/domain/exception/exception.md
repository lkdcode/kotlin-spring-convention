# domain/exception — Domain Exception

도메인 불변식 위반, 비즈니스 규칙 위반을 표현하는 예외.

```kotlin
// 도메인 예외 기반 클래스
sealed class DomainException(message: String) : RuntimeException(message)

// 구체 예외 — 도메인 언어로 네이밍
class OrderAlreadyConfirmedException(orderId: OrderId) :
    DomainException("이미 확정된 주문입니다: ${orderId.value}")

class InsufficientStockException(productId: ProductId, requested: Int, available: Int) :
    DomainException("재고 부족 — 상품: ${productId.value}, 요청: $requested, 가용: $available")

class InvalidOrderStatusTransitionException(from: OrderStatus, to: OrderStatus) :
    DomainException("유효하지 않은 상태 전이: $from → $to")
```

## Application 계층에서 변환

```kotlin
// Application/Adapter 경계에서 DomainException → ApiException 변환
@RestControllerAdvice
class DomainExceptionHandler {

    @ExceptionHandler(OrderAlreadyConfirmedException::class)
    fun handle(e: OrderAlreadyConfirmedException): ResponseEntity<ApiResponse<Nothing>> =
        ApiResponse.clientError(ApiResponseCode.ORDER_ALREADY_CONFIRMED, e.message)
}
```

## Domain Exception vs Application Exception 구분

| | Domain Exception | Application Exception |
|---|---|---|
| 위치 | `domain/exception/` | `framework/api/exception/` |
| 발생 | Entity / Domain Service 내부 | Validator / Guard |
| 의미 | 불변식 위반 | 유스케이스 실패 |
| 예시 | `InsufficientStockException` | `ApiException(NOT_FOUND)` |

## 규칙
- `sealed class` 로 도메인 예외 계층 구성
- 메시지에 도메인 맥락 포함 (어떤 ID, 어떤 값)
- 도메인 계층은 HTTP 상태코드, ApiResponse 를 모름
- Adapter 경계에서 변환 처리
