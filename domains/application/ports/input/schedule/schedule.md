# application/ports/input/schedule — *ScheduleInput.kt (interface)

스케줄러가 호출하는 인바운드 포트. UseCase 와 동일한 역할.

```kotlin
interface FooScheduleInput {
    fun execute()
}
```

## 구현체 네이밍

| 인터페이스 | 구현체 |
|---|---|
| `FooScheduleInput` | `FooScheduleService` |

```kotlin
@Service
@Transactional
class FooScheduleService(
    private val commandPort: FooCommandPort,
) : FooScheduleInput {

    override fun execute() {
        commandPort.deleteExpired()
    }
}
```

## 규칙

- 메서드명 `execute()` 고정 — 스케줄러는 단일 작업
- 구현체는 `application/services/schedule/` 에 위치
- `application/ports/output/` 의 포트만 주입 — adapter 계층 코드 import 금지
