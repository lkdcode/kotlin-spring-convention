# application/ports/input/query/common — Query 공통 규약

`*QueryUsecase.kt` 의 공통 입출력 계약. 모든 도메인의 Query 에서 공유.

**`domain/` 은 이 파일을 참조하지 않는다.**

## Pagination

```kotlin
@JsonPropertyOrder("totalItems", "totalPages", "currentPage", "perPage")
class Pagination private constructor(
    val totalItems: Long,
    val totalPages: Int,
    val currentPage: Int,
    val perPage: Int,
) {
    companion object {
        fun of(page: Page<*>): Pagination =
            Pagination(
                totalItems = page.totalElements,
                totalPages = page.totalPages,
                currentPage = page.number + 1,
                perPage = page.size,
            )
    }
}
```

## ParamConditionList (범용 검색 조건)

도메인 전용 Condition 이 과하거나, 동적 쿼리 조건이 필요할 때 사용.
내부는 `String` 으로 저장 — 외부 API 는 `Enum<*>` 으로 타입 안전성 확보.

```kotlin
data class ParamCondition(
    val key: String,
    val value: String,
) {
    fun isNotEmpty(): Boolean = value.isNotEmpty()
}

data class ParamConditionList(
    val list: List<ParamCondition>,
) {
    companion object {
        fun of(vararg pairs: Pair<Enum<*>, String>): ParamConditionList =
            ParamConditionList(pairs.map { ParamCondition(it.first.name, it.second) })
    }

    fun valueOf(key: Enum<*>): String? =
        list.find { it.key == key.name }?.value?.takeIf { it.isNotEmpty() }
}
```

### 전체 흐름

**1. enum key 정의** (`ports/input/query/` 도메인 위치)
```kotlin
enum class FooConditionKey { NAME, CODE, STATUS }
```

**2. Controller → ParamConditionList 변환** (`adapter/input/`)
```kotlin
data class FooQueryString(
    val name: String? = null,
    val code: String? = null,
    val status: String? = null,
) {
    fun toCondition() = ParamConditionList.of(
        *listOfNotNull(
            name?.let { FooConditionKey.NAME to it },
            code?.let { FooConditionKey.CODE to it },
            status?.let { FooConditionKey.STATUS to it },
        ).toTypedArray()
    )
}
```

**3. UseCase → Service → QueryPort** (그대로 전달)
```kotlin
interface FetchFooUsecase {
    fun fetchList(condition: ParamConditionList, pageable: Pageable): FetchFooDTOList
}
```

**4. Adapter — 쿼리 빌더별 조건 적용**

jOOQ:
```kotlin
private fun buildCondition(condition: ParamConditionList): Condition {
    var result = DSL.trueCondition()
    condition.valueOf(FooConditionKey.NAME)?.let { result = result.and(FOO.NAME.containsIgnoreCase(it)) }
    condition.valueOf(FooConditionKey.STATUS)?.let { result = result.and(FOO.STATUS.eq(it)) }
    return result
}
```

QueryDSL:
```kotlin
private fun buildPredicate(condition: ParamConditionList) =
    BooleanBuilder().apply {
        condition.valueOf(FooConditionKey.NAME)?.let { and(foo.name.containsIgnoreCase(it)) }
        condition.valueOf(FooConditionKey.STATUS)?.let { and(foo.status.eq(it)) }
    }
```

JPA Specification:
```kotlin
private fun buildSpec(condition: ParamConditionList) =
    Specification<FooJpaEntity> { root, _, cb ->
        val predicates = mutableListOf<Predicate>()
        condition.valueOf(FooConditionKey.NAME)?.let {
            predicates.add(cb.like(cb.lower(root.get("name")), "%${it.lowercase()}%"))
        }
        condition.valueOf(FooConditionKey.STATUS)?.let {
            predicates.add(cb.equal(root.get<String>("status"), it))
        }
        cb.and(*predicates.toTypedArray())
    }
```

`valueOf(EnumKey)` 가 `null` 이면 조건 건너뜀 — jOOQ / QueryDSL / JPA Specification 모두 동일한 패턴으로 호환.

## 규칙

- `domain/` 에서 참조 금지
- enum key 는 각 도메인에서 정의 — `ParamConditionList` 는 enum 을 모름
- 도메인 전용 Condition(`FooSearchCondition`)이 명확하면 `ParamConditionList` 불필요
