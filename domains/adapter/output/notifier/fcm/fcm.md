# adapter/output/notifier/fcm — *NotifierAdapter.kt

포트: `application/ports/output/notifier/`
FCM 을 통해 특정 디바이스로 푸시 알림 발송.

```kotlin
@Component
class FooNotifierAdapter(
    private val fcmService: FcmService,
) : FooNotifier {

    override fun notify(
        deviceTokenList: List<String>,
        model: FooModel,
    ) {
        fcmService.pushDataOnly(
            deviceTokenList,
            FcmPushModel.of(ChannelType.FOO, OperationType.PUB, model),
        )
    }

    override fun notify(
        deviceToken: String,
        model: FooModel,
    ) {
        fcmService.pushDataOnly(
            deviceToken,
            FcmPushModel.of(ChannelType.FOO, OperationType.PUB, model),
        )
    }
}
```

## 규칙

- 비즈니스 로직 작성 금지 — `fcmService` 호출만
- 포트 인터페이스에만 의존 (`rules/dependency.md` 참조)
