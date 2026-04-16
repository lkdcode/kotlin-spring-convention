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
