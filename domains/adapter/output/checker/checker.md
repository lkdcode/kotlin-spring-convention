# adapter/output/checker — *CheckerAdapter.kt

반환 타입 `Boolean`, **throw 없음**

```kotlin
@Service
class FooCheckerAdapter(
    private val dsl: DSLContext,
) : FooChecker {

    override fun isDuplicate(code: String): Boolean =
        dsl.fetchExists(
            dsl.selectOne().from(FOO)
                .where(FOO.CODE.eq(code))
                .and(FOO.IS_DELETED.isFalse)
        )

    companion object {
        val FOO = Tables.MST_FOO
    }
}
```

## 규칙
- 반환 타입 `Boolean` — throw 없음
- 호출부에서 결과를 받아 직접 분기 처리
- jOOQ `DSLContext` 사용
- 테이블 상수는 `companion object` 에 선언
- 포트 위치: `application/ports/output/checker/`
