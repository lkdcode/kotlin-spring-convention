# application/services/query — *QueryService.kt (구현체)

```kotlin
// FetchFooUsecase → FooQueryService
@Service
@Transactional(readOnly = true)
class FooQueryService(
    private val queryPort: FooQueryPort,
) : FetchFooUsecase {

    override fun getById(id: FooId): FetchFooDTO =
        queryPort.getById(id)

    override fun fetchList(condition: FooSearchCondition, pageable: Pageable): FetchFooDTOList =
        queryPort.fetchList(condition, pageable)

    override fun fetchLookupList(): List<FooLookupDTO> =
        queryPort.fetchLookupList()
}
```

## 규칙

- `@Transactional(readOnly = true)` 필수
- `application/ports/output/` 의 포트만 주입 — **adapter 계층 코드 import 절대 금지**
- 타 도메인의 `ports/output/*` 참조 가능 — 도메인 간 Checker 등 포트 인터페이스를 통해 활용
- `domain` 사용 적극 활용 — `spec`, `policy`, `service` 등
- 단건 null 반환 금지 — 없으면 throw (`queryPort` 에서 처리)

## 위치 참조

| 종류 | 위치 |
|---|---|
| 응답 DTO | `ports/input/query/dto/` |
| 검색 조건 / Pagination / ParamConditionList | `ports/input/query/common/` |
