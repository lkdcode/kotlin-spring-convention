# rules/dependency — 의존성 규칙

계층 무관하게 모든 코드에 적용.

---

## 핵심 원칙

**구현체가 아닌 인터페이스(포트)에 의존한다.**

```kotlin
// ✅ 포트 인터페이스 의존
class FooEventHandler(
    private val commandPort: FooCommandPort,       // interface
    private val publisher: FooPublisher,           // interface
)

// ❌ 구현체 직접 의존
class FooEventHandler(
    private val commandAdapter: FooCommandAdapter, // 구현체
    private val publishAdapter: FooPublishAdapter, // 구현체
)
```

---

## Adapter 계층 의존 규칙

| 계층 | 의존 허용 | 의존 금지 |
|---|---|---|
| `adapter/input/*` | `application/ports/input/` (UseCase) | 다른 adapter 구현체 |
| `adapter/input/*` | `application/ports/output/` (포트 인터페이스) | `adapter/output/` 구현체 |
| `adapter/output/*` | `application/ports/output/` (구현 대상) | `adapter/input/` |
| `adapter/output/*` | `domain/` | 다른 `adapter/output/` 구현체 |

---

## 의존 방향

```
adapter/input
    ↓
application/ports/input   (UseCase interface)
application/ports/output  (Port interface)
    ↓
adapter/output
    ↓
infrastructure (JPA, jOOQ, Kafka producer 등)
```

adapter 간 직접 의존 금지 — 반드시 포트 인터페이스를 통해 간접 의존.
