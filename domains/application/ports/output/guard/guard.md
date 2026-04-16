# application/ports/output/guard — *Guard.kt (interface)

## 공통 규칙 (ports/output 전체)

- 타 도메인의 동일 포트와 중복 작성 금지 — 단, 도메인 정책이 다른 경우 분리 고려
- 단일 책임 원칙 준수 — 하나의 액터 기준으로 책임 분리
- 구현체는 `adapter/output/validator/` (GuardAdapter) 에 동일 구조로 위치
- 네이밍: `XxxGuard`

## Guard 규칙

- 함수명 접두사 **`requireXxx`**
- 반환 타입 `Unit` — **Boolean 리턴 금지**
- **무조건 exception throw** — 검증 실패 시 `ApiException`
- **상태/권한 기반 작업 허용 여부** 검사 전용

```kotlin
interface FooGuard {
    fun requireAdminRole(userId: Long)              // 권한 기반 작업 허용
    fun requireCancellable(orderId: Long)           // 현재 상태에서 취소 가능 여부
    fun requireCanTransition(from: String, to: String) // 상태 전이 허용 여부
}
```

## Validator 와의 차이

기능은 동일(Unit 반환, 무조건 throw). **질문의 방향이 다르다.**

```
Guard     → "이 상태/권한에서 이 작업이 허용되는가?"
Validator → "이 데이터/값이 조건을 만족하는가?"
```

| Guard 로 작성할 때 | Validator 로 작성할 때 |
|---|---|
| 역할/권한 기반 작업 허용 여부 | 입력값 크기, 형식, 범위 제한 |
| 현재 상태에서 전이 가능 여부 | 중복 여부, 존재 여부 |
| 특정 작업의 사전 조건 검사 | 생성 요청 데이터의 조건 검사 |
| `requireAdminRole` | `validateContentLength` |
| `requireCancellable` | `validateMinimumMembers` |

> 애매하면 **Validator 선택**. 함수명 `requireXxx` 에 자연스럽게 맞으면 Guard.
