# application/ports/output/query — *QueryPort.kt (interface)

## 공통 규칙 (ports/output 전체)

- 타 도메인의 동일 포트와 중복 작성 금지 — 단, 도메인 정책이 다른 경우 분리 고려
- 단일 책임 원칙 준수 — 하나의 액터 기준으로 책임 분리
- 구현체는 `adapter/output/query/` 에 동일 구조로 위치
- 네이밍: `XxxQueryPort`

## QueryPort 규칙

```kotlin
interface FooQueryPort {
    // 단건 — get
    fun getById(id: FooId): FetchFooDTO

    // 다건 — fetch
    fun fetchList(condition: FooSearchCondition, pageable: Pageable): FetchFooDTOList
    fun fetchLookupList(): List<FooLookupDTO>
    fun fetchAllByUserId(userId: Long): List<FetchFooDTO>
}
```

## 함수 네이밍

| prefix | 대상 | 예시 |
|---|---|---|
| `get` | **단건** — 반드시 존재, 없으면 throw | `getById`, `getByCode` |
| `fetch` | **다건** — 목록, 페이징, 전체 | `fetchList`, `fetchLookupList`, `fetchAllByUserId` |

## null 반환 정책

- **단건(`get`) null 반환 금지** — 존재 전제, 없으면 throw
- 다건(`fetch`) 빈 배열 반환 허용 — `FetchFooDTOList(items = emptyList(), ...)`
- null 반환은 사용자 명시적 요청이 있는 경우에만 예외적으로 허용

## 검색 조건 — Condition 객체

### 권장: 도메인 전용 Condition 객체

```kotlin
// 타입 안전, 컴파일 타임 오타 검출
data class FooSearchCondition(
    val name: String? = null,
    val code: String? = null,
)
```

### 대안: key-value 방식 — enum key 로 타입 안전성 확보

```kotlin
// String key 오타 방지 — enum 으로 key 고정
enum class FooConditionKey { NAME, CODE, STATUS }

data class ParamCondition(
    val key: FooConditionKey,
    val value: String,
) {
    fun isNotEmpty(): Boolean = value.isNotEmpty()
}

data class ParamConditionList(
    val list: List<ParamCondition>,
) {
    fun valueOf(key: FooConditionKey): String? =
        list.find { it.key == key }?.value?.takeIf { it.isNotEmpty() }
}

// 사용
val condition = ParamConditionList(listOf(
    ParamCondition(FooConditionKey.NAME, "홍길동"),
    ParamCondition(FooConditionKey.STATUS, "ACTIVE"),
))
```

> String key 방식(`ParamConditionList(key: String)`)은 오타를 컴파일 타임에 잡지 못하므로 사용 금지.

## Pageable / Pagination

`org.springframework.data.domain.Pageable` 사용 허용.

```kotlin
data class FetchFooDTOList(
    val items: List<FetchFooDTO>,
    val pagination: Pagination,    // Page<*> 를 application 계층에서 격리
)
```
