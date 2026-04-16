# application/ports/input/query — *Usecase.kt (interface)

```kotlin
interface FetchFooUsecase {
    fun getById(id: FooId): FetchFooDTO                                          // 단건
    fun fetchList(condition: FooSearchCondition, pageable: Pageable): FetchFooDTOList  // 다건
    fun fetchLookupList(): List<FooLookupDTO>                                    // 전체 목록
}
```

## 규칙

- 인터페이스 선언만
- Query Usecase 는 하나로 통합 (`Fetch*Usecase`)
- 메서드는 조회 목적에 따라 복수 선언 가능
- 단건 `get`, 다건 `fetch` — 상세 네이밍은 `application/ports/output/query/query.md` 참조

## 구현체 네이밍

| 인터페이스 | 구현체 |
|---|---|
| `FetchFooUsecase` | `FooQueryService` |

## 반환 타입

- 조회 응답은 DTO 반환 — `application/ports/input/query/dto/dto.md` 참조
- null 반환 금지 — 단건은 없으면 throw, 다건은 빈 배열 반환

## 위치 참조

| 종류 | 위치 |
|---|---|
| 응답 DTO | `ports/input/query/dto/` |
| 검색 조건 / Pagination / ParamConditionList | `ports/input/query/common/` |
