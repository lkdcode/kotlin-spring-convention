# adapter/input/rest/query — *QueryApi.kt

```kotlin
@RestController
@Tag(name = "NNN-NNN. {도메인} 관리 API", description = "NNN-NNN. {도메인} 관리 API")
class FooQueryApi(
    private val fetchUsecase: FetchFooUsecase,
) {
    @Operation(summary = "Foo 단건 조회")
    @GetMapping("/api/foos/{id}")
    fun getById(
        @PathVariable id: Long,
    ): ApiResponse<FetchFooDTO> =
        ApiResponse(fetchUsecase.getById(FooId(id)))

    @Operation(summary = "Foo 목록 조회")
    @GetMapping("/api/foos")
    fun fetchList(
        @ParameterObject
        @PageableDefault(size = 15, page = 0, sort = ["name"], direction = ASC) pageable: Pageable,
        @ParameterObject
        @ModelAttribute queryString: FooQueryString,
    ): ApiResponse<FetchFooDTOList> =
        ApiResponse(fetchUsecase.fetchList(queryString.toCondition(), pageable))

    @Operation(summary = "Foo 룩업 목록")
    @GetMapping("/api/foos/lookup")
    fun fetchLookupList(): ApiResponse<List<FooLookupDTO>> =
        ApiResponse(fetchUsecase.fetchLookupList())
}

// QueryApi 파일 하단에 같이 선언
data class FooQueryString(
    val name: String? = null,
    val code: String? = null,
) {
    fun toCondition() = ParamConditionList.of(
        *listOfNotNull(
            name?.let { FooConditionKey.NAME to it },
            code?.let { FooConditionKey.CODE to it },
        ).toTypedArray()
    )
}
```

## 규칙

- 클래스명: `*QueryApi`
- `FetchUsecase` 하나만 주입 — `fetchUsecase` 네이밍
- 반환: `ApiResponse<T>` — 상세는 `rest/common/common.md` 참조
- `QueryString` 은 QueryApi 파일 하단에 같이 선언
- `QueryString.toCondition()` 으로 `ParamConditionList` 변환
- 페이징 기본값: `size=15, page=0`
- `@RequestMapping` 사용 절대 금지 — 각 메서드에 풀 경로 작성
- 엔드포인트 네이밍: `/api` 로 시작 후 도메인 표현
- UseCase 호출 후 한 줄 개행 후 `return` 작성
- 비즈니스 로직 작성 금지 — UseCase 호출만
- `@Operation(summary = "...")` 필수
- lookup 엔드포인트: `/api/{domain}/lookup`
