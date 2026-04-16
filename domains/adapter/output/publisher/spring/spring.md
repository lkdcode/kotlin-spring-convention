# adapter/output/publisher/spring — *PublisherAdapter.kt

포트: `application/ports/output/publisher/`
Spring `ApplicationEventPublisher` 로 도메인 이벤트 발행.
`adapter/input/event/spring/` 의 `@TransactionalEventListener` 핸들러가 수신.

```kotlin
@Component
class SpringDomainEventPublisherAdapter(
    private val applicationEventPublisher: ApplicationEventPublisher,
) : DomainEventPublisher {

    override fun publish(
        event: DomainEvent,
    ) {
        applicationEventPublisher.publishEvent(event)
    }
}
```

## 규칙

- 비즈니스 로직 작성 금지 — `publishEvent()` 호출만
- 포트 인터페이스에만 의존 (`rules/dependency.md` 참조)
