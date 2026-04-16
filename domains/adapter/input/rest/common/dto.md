# adapter/input/rest/common/dto — Request / Response / Mapper

`adapter/input/rest/` 의 모든 Command / Query 에 동일하게 적용.

---

## Request DTO

HTTP 요청 매핑 전용. `@field:` 어노테이션과 `toModel()` 은 여기에.

```kotlin
data class CreateFooRequest(
    @field:Schema(description = "부품명", example = "SUS304")
    @field:NotBlank
    val name: String,

    @field:Schema(description = "부품 코드", example = "ITEM-001")
    @field:NotBlank
    val code: String,

    val description: String? = null,
) {
    fun toModel() = CreateFooModel(
        name = name,
        code = code,
        description = description,
    )
}
```

QueryString 도 동일한 패턴:

```kotlin
data class FooQueryString(
    val name: String? = null,
    val status: String? = null,
) {
    fun toCondition() = ParamConditionList.of(
        *listOfNotNull(
            name?.let { FooConditionKey.NAME to it },
            status?.let { FooConditionKey.STATUS to it },
        ).toTypedArray()
    )
}
```

---

## Response DTO

Command 응답은 대부분 `ApiResponse<String>` 으로 충분.
Query 응답은 application DTO(`FetchFooDTO`) 를 그대로 반환.
별도 Response DTO 는 구조가 다를 때만 선언.

```kotlin
// 대부분 — 별도 DTO 불필요
fun create(...): ApiResponse<String>
fun fetchList(...): ApiResponse<FetchFooDTOList>

// 생성 후 데이터 반환이 필요한 경우
data class CreateFooResponse(
    val id: Long,
    val code: String,
)
fun create(...): ApiResponse<CreateFooResponse>
```

---

## 규칙

- `@field:NotBlank`, `@field:Schema` 등 HTTP 관심사는 Request DTO 에만
- application model(`CreateFooModel`) 을 `@RequestBody` 로 직접 수신 금지
- 변환: `toModel()` / `toCondition()` — Request DTO 내부에서 처리
- Response DTO: application DTO 와 구조가 동일하면 중복 선언 금지
