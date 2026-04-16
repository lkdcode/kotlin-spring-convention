# application/ports/output/checker — *Checker.kt (interface)

## 공통 규칙 (ports/output 전체)

- 타 도메인의 동일 포트와 중복 작성 금지 — 단, 도메인 정책이 다른 경우 분리 고려
- 단일 책임 원칙 준수 — 하나의 액터 기준으로 책임 분리
- 구현체는 `adapter/output/checker/` 에 동일 구조로 위치
- 네이밍: `XxxChecker`

## Checker 규칙

- 함수명 접두사 **`isXxx`**, **`hasXxx`**, **`canXxx`** 등 Boolean 의미의 동사
- 반환 타입 `Boolean` — 호출부에서 결과를 받아 직접 분기 처리
- **exception 금지**

```kotlin
interface FooChecker {
    fun isDuplicate(code: String): Boolean
    fun isExists(id: Long): Boolean
    fun hasPermission(userId: Long, resourceId: Long): Boolean
    fun canDelete(id: Long): Boolean
}
```

## Checker vs Validator vs Guard — 선택 기준

| | Checker | Validator | Guard |
|---|---|---|---|
| 반환 | `Boolean` | `Unit` | `Unit` |
| 실패 시 | **호출부가 분기 처리** | throw | throw |
| 용도 | 결과를 받아 후속 작업 이어감 | 데이터 유효성 검증 | 상태/권한 기반 작업 허용 |
| 접두사 | `is`, `has`, `can` | `validate` | `require` |

### 언제 Checker 를 쓰는가

**Boolean 을 받아서 호출부에서 작업을 이어가야 할 때.** Validator/Guard 는 실패하면 즉시 예외로 중단되지만, Checker 는 결과에 따라 **다른 경로로 처리를 계속**해야 할 때 사용한다.

```kotlin
// ✅ Checker — 결과에 따라 분기 후 작업 계속
if (fooChecker.isDuplicate(model.code)) {
    commandPort.update(model)      // 중복이면 수정
} else {
    commandPort.create(model)      // 아니면 생성
}

// ✅ Validator — 실패하면 중단, 성공하면 다음 단계
fooValidator.validateDuplicate(model.code)  // 중복이면 throw → 여기서 끝
commandPort.create(model)                   // 중복 아니면 생성
```
