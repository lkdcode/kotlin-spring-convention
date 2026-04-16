# adapter/input/event/spring — *EventHandler.kt

Spring ApplicationEventPublisher 로 발행된 이벤트를 수신.
트랜잭션과 연동되므로 커밋 후 실행 권장.

```kotlin
@Component
class FooEventHandler(
    private val someUsecase: SomeUsecase,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun on(
        event: FooCreatedEvent,
    ) {
        someUsecase.execute(event.fooId)
    }
}
```

## @TransactionalEventListener vs @EventListener

| | `@TransactionalEventListener` | `@EventListener` |
|---|---|---|
| 실행 시점 | 트랜잭션 커밋 후 | 이벤트 발행 즉시 |
| 트랜잭션 롤백 | 영향 없음 | 함께 롤백 |
| 사용 시점 | DB 커밋이 보장되어야 할 때 | 트랜잭션 무관한 처리 |

## 공통 규칙 (event 계층 전체)

- 비즈니스 로직 직접 작성 금지
- UseCase(interface) 만 호출 — event 전용 UseCase 가 없으면 생성
- 트랜잭션은 UseCase 에서만 처리
- 포트 인터페이스에만 의존 — 구현체 직접 import 금지 (`rules/dependency.md` 참조)

## Spring Event 규칙

- 클래스명: `*EventHandler`
- `@TransactionalEventListener(phase = AFTER_COMMIT)` 권장
- 실패 시 트랜잭션 롤백 불가 — 별도 보상 처리 필요
