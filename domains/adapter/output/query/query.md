# adapter/output/query — *QueryAdapter.kt

## *QueryAdapter.kt — jOOQ

```kotlin
@Service
class FooQueryAdapter(
    private val dsl: DSLContext,
) : FooQueryPort {

    override fun fetchList(condition: FooSearchCondition, pageable: Pageable): FetchFooDTOList {
        val query = dsl
            .select(
                FOO.ID, FOO.NAME, FOO.CODE, FOO.IS_DELETED,
                FOO.CREATED_AT, CREATED_BY_USER.NAME.`as`("createdByName"),
            )
            .from(FOO)
            .leftJoin(CREATED_BY_USER).on(CREATED_BY_USER.ID.eq(FOO.CREATED_BY))
            .where(buildCondition(condition))

        val totalSize = dsl.fetchCount(query)

        val items = query
            .orderBy(buildOrderBy(pageable))
            .limit(pageable.pageSize)
            .offset(pageable.offset)
            .fetchInto(FetchFooDTO::class.java)

        return FetchFooDTOList(items = items, pagination = Pagination.of(pageable, totalSize.toLong()))
    }

    override fun fetchLookupList(): List<FooLookupDTO> =
        dsl.select(FOO.ID, FOO.NAME)
            .from(FOO)
            .where(FOO.IS_DELETED.isFalse)
            .orderBy(FOO.NAME.asc())
            .fetchInto(FooLookupDTO::class.java)

    private fun buildCondition(condition: FooSearchCondition): Condition {
        var result: Condition = DSL.trueCondition()
        condition.name?.takeIf { it.isNotBlank() }?.let {
            result = result.and(FOO.NAME.containsIgnoreCase(it))
        }
        condition.code?.takeIf { it.isNotBlank() }?.let {
            result = result.and(FOO.CODE.containsIgnoreCase(it))
        }
        return result
    }

    private fun buildOrderBy(pageable: Pageable): List<SortField<*>> =
        pageable.sort.map { order ->
            val field = when (order.property) {
                "name" -> FOO.NAME
                "code" -> FOO.CODE
                else   -> FOO.CREATED_AT
            }
            if (order.isAscending) field.asc() else field.desc()
        }.ifEmpty { listOf(FOO.CREATED_AT.desc()) }

    companion object {
        val FOO = Tables.MST_FOO
        val CREATED_BY_USER = Tables.MST_USER.`as`("createdByUser")
    }
}
```

## 규칙
- 복잡한 조회, 페이징, JOIN, 조건 검색 → jOOQ `DSLContext`
- 단순 단건 조회 (ID 조회, exists 확인) → JPA Repository 확장함수 허용
- 검색 조건은 `buildCondition()` 으로 분리
- 정렬은 `buildOrderBy()` 으로 분리
- `fetchInto(DTO::class.java)` 로 컬럼명 매핑 (DTO 필드명 = DB 컬럼 camelCase)
- 테이블 상수는 `companion object` 에 선언
