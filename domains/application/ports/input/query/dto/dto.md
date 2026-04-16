# application/ports/input/query/dto — Query DTO

UseCase 의 조회 응답 계약. UseCase 와 함께 변하므로 같은 위치에 선언.

```kotlin
// 단건 조회 응답
data class FetchFooDTO(
    val id: Long,
    val name: String,
    val code: String,
    val createdAt: Instant,
)

// 페이징 목록 래퍼
data class FetchFooDTOList(
    val items: List<FetchFooDTO>,
    val pagination: Pagination,
)

// 드롭다운 등 경량 조회
data class FooLookupDTO(
    val id: Long,
    val name: String,
)
```

## 규칙

- 조회 응답 전용 — Command Model 과 혼용 금지
- 도메인 VO 래핑 불필요 — 조회 응답은 단순 DTO 로 선언
- **JPA Entity, adapter 계층 타입 사용 금지**
- 페이징 응답은 `FetchXxxDTOList` 로 래핑 (`items` + `pagination`)
- 드롭다운 등 경량 응답은 `XxxLookupDTO` 로 선언 (id + 표시명만)
