# application/ports/output/notifier — *Notifier.kt (interface)

**알림 전용** — 특정 수신자(디바이스/사용자)에게 발송.
FCM, APNS, SMS 등.
구현체: `adapter/output/notifier/fcm/`

```kotlin
interface FooNotifier {
    fun notify(deviceTokenList: List<String>, model: FooModel)
    fun notify(deviceToken: String, model: FooModel)
}
```

## 규칙

- 인터페이스에 fcm / apns 등 기술 용어 포함 금지
- 단일 책임 — 특정 수신자 알림 역할만
- 브로드캐스트 → `publisher/`, 이벤트 발행 → `producer/` 로 분리
