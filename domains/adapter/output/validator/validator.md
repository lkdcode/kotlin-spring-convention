# adapter/output/validator — *ValidatorAdapter.kt / *GuardAdapter.kt

## *ValidatorAdapter.kt

포트: `application/ports/output/validator/`
반환 타입 `Unit`, 조건 불충족 시 **throw**. 함수명 접두사 `validateXxx`.

```kotlin
@Service
class FooValidatorAdapter(
    private val dsl: DSLContext,
) : FooValidator {

    override fun validateDuplicate(code: String) {
        val exists = dsl.fetchExists(
            dsl.selectOne().from(FOO)
                .where(FOO.CODE.eq(code))
                .and(FOO.IS_DELETED.isFalse)
        )
        if (exists) throw ApiException(ApiResponseCode.FOO_DUPLICATE_CODE)
    }

    override fun validateExists(id: Long) {
        val exists = dsl.fetchExists(
            dsl.selectOne().from(FOO).where(FOO.ID.eq(id))
        )
        if (!exists) throw ApiException(ApiResponseCode.FOO_NOT_FOUND)
    }

    companion object {
        val FOO = Tables.MST_FOO
    }
}
```

## *GuardAdapter.kt

포트: `application/ports/output/guard/`
반환 타입 `Unit`, 비즈니스 규칙 위반 시 **throw**. 함수명 접두사 `requireXxx`. 내부적으로 `domain/spec` 활용.

```kotlin
@Service
class FooGuardAdapter(
    private val dsl: DSLContext,
) : FooGuard {

    override fun requireCanDelete(id: Long) {
        val status = dsl.select(FOO.STATUS).from(FOO)
            .where(FOO.ID.eq(id))
            .fetchOneInto(String::class.java)
            ?: throw ApiException(ApiResponseCode.FOO_NOT_FOUND)

        if (!FooStatusSpec.canDelete(status))
            throw ApiException(ApiResponseCode.FOO_CANNOT_DELETE)
    }

    companion object {
        val FOO = Tables.MST_FOO
    }
}
```

## Validator vs Guard

| | Validator | Guard |
|---|---|---|
| 함수명 접두사 | `validateXxx` | `requireXxx` |
| 대상 | 데이터 유효성 (중복, 존재 여부) | 비즈니스 규칙 (상태 전이, 삭제 조건) |
| 의존 | jOOQ DSLContext | jOOQ DSLContext + domain/spec |

## 규칙

- 반환 타입 `Unit` — 위반 시 `ApiException` throw
- jOOQ `DSLContext` 사용
- Guard 는 내부적으로 `domain/spec` 의 판단 함수 활용
- 테이블 상수는 `companion object` 에 선언
- 포트 위치: `application/ports/output/validator/`, `application/ports/output/guard/`
