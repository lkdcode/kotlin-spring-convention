# application/ports/output/publisher — *Publisher.kt (interface)

**브로드캐스트 전용** — 연결된 수신자 전체에게 전파.
WebSocket 브로드캐스트, Spring Domain Event 발행.
구현체: `adapter/output/publisher/socket/`, `adapter/output/publisher/spring/`

```kotlin
// WebSocket 브로드캐스트
interface FooPublisher {
    fun publish(roomId: FooRoomId, model: FooModel)
    fun publishWithoutMe(userId: Long, roomId: FooRoomId, model: FooModel)
    fun delete(roomId: FooRoomId, model: FooModel)
}

// Spring Domain Event 발행
interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}
```

## 규칙

- 인터페이스에 socket / spring 등 기술 용어 포함 금지
- 단일 책임 — 브로드캐스트 역할만
- 알림 발송 → `notifier/`, 이벤트 발행 → `producer/` 로 분리
