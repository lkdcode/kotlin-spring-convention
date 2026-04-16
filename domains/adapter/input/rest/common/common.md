# adapter/input/rest/common — REST 공통 규약

`adapter/input/rest/` 의 모든 Controller 가 따르는 공통 계약.
`web/`, `event/`, `schedule/` 등 다른 adapter/input 에서는 사용하지 않음.

## ApiResponse — typealias + factory

### 구현체

```kotlin
// 1. Response body — framework/api/response/ApiBody 와 동일한 구조
@JsonPropertyOrder("success", "code", "message", "payload")
data class ApiBody<T>(
    val success: Boolean,
    val code: String? = null,
    val message: String,
    val payload: T? = null,
)

// 2. Return type alias
typealias ApiResponse<T> = ResponseEntity<ApiBody<T>>

// 3. Factory functions (최상위 함수 — typealias 와 이름 공유 가능)
fun <T> ApiResponse(
    payload: T,
    code: ApiResponseCode = ApiResponseCode.OK,
): ApiResponse<T> =
    ResponseEntity.ok(
        ApiBody(success = true, code = code.code, message = code.message, payload = payload)
    )

fun ApiResponseCreated(message: String): ApiResponse<String> =
    ResponseEntity
        .status(HttpStatus.CREATED)
        .body(ApiBody(success = true, code = ApiResponseCode.CREATED.code, message = message))

fun ApiResponseUpdated(message: String): ApiResponse<String> =
    ResponseEntity.ok(
        ApiBody(success = true, code = ApiResponseCode.UPDATED.code, message = message)
    )

fun ApiResponseDeleted(message: String): ApiResponse<String> =
    ResponseEntity.ok(
        ApiBody(success = true, code = ApiResponseCode.DELETED.code, message = message)
    )

fun ApiResponseError(
    apiCode: ApiResponseCode,
    message: String? = null,
): ApiResponse<Nothing> =
    ResponseEntity
        .status(apiCode.status)
        .body(ApiBody(success = false, code = apiCode.code, message = message ?: apiCode.message))
```

### CommandApi 적용

```kotlin
@RestController
class FooCommandApi(
    private val createUsecase: CreateFooUsecase,
    private val deleteUsecase: DeleteFooUsecase,
) {
    @PostMapping("/api/foos")
    fun create(
        @Valid @RequestBody request: CreateFooRequest,
    ): ApiResponse<String> {
        createUsecase.execute(request.toModel())

        return ApiResponseCreated("Foo 생성에 성공했습니다.")
    }

    @DeleteMapping("/api/foos")
    fun delete(
        @RequestBody ids: List<Long>,
    ): ApiResponse<String> {
        deleteUsecase.execute(ids)

        return ApiResponseDeleted("Foo 삭제에 성공했습니다.")
    }
}
```

### QueryApi 적용

```kotlin
@RestController
class FooQueryApi(private val fetchUsecase: FetchFooUsecase) {

    @GetMapping("/api/foos/{id}")
    fun getById(
        @PathVariable id: Long,
    ): ApiResponse<FetchFooDTO> =
        ApiResponse(fetchUsecase.getById(FooId(id)))

    @GetMapping("/api/foos")
    fun fetchList(
        @ParameterObject
        @PageableDefault(size = 15) pageable: Pageable,
        @ParameterObject
        @ModelAttribute queryString: FooQueryString,
    ): ApiResponse<FetchFooDTOList> =
        ApiResponse(fetchUsecase.fetchList(queryString.toCondition(), pageable))

    @GetMapping("/api/foos/lookup")
    fun fetchLookupList(): ApiResponse<List<FooLookupDTO>> =
        ApiResponse(fetchUsecase.fetchLookupList())
}
```

### 반환 타입 비교

| | Before (framework 직접 사용) | After (typealias) |
|---|---|---|
| 반환 타입 | `ResponseEntity<ApiBody<T>>` | `ApiResponse<T>` |
| 생성 응답 | `ResponseEntity.status(201).body(ApiBody(...))` | `ApiResponseCreated("...")` |
| 조회 응답 | `ResponseEntity.ok(ApiBody(...))` | `ApiResponse(payload)` |

> `ApiBody` 클래스는 `framework/api/response/` 에 위치.
> `ApiResponse` typealias 와 factory 함수는 `adapter/input/rest/common/` 에 선언.
