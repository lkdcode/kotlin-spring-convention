# adapter/input/rest/common/swagger — Swagger 문서 작성 양식

## 어노테이션 한눈에 보기

| 어노테이션 | 위치 | 필수 | 설명 |
|---|---|---|---|
| `@Tag` | 클래스 | ✅ | API 그룹명 |
| `@Operation` | 메서드 | ✅ | 엔드포인트 요약/설명 |
| `@Parameter` | 파라미터 | ❌ | PathVariable, QueryParam 설명 |
| `@Schema` | DTO 필드 | ❌ | 필드 설명, 예시값 |
| `@ApiResponses` | 메서드 | ❌ | 응답 코드 명시 |

---

## @Tag — 컨트롤러 단위

```kotlin
@RestController
@Tag(name = "001-005. 부품 API", description = "001-005. 부품 API")
class FooCommandApi(...)
```

| 속성 | 설명 | 형식 |
|---|---|---|
| `name` | API 그룹명 | `"NNN-NNN. {도메인} API"` — 메뉴 순서 기준 |
| `description` | 그룹 설명 | `name` 과 동일하게 작성 |

---

## @Operation — 메서드 단위

```kotlin
@Operation(
    summary = "부품 생성",
    description = """
신규 부품을 생성합니다.  
- 부품 코드는 중복 불가  
- 삭제된 부품 코드는 재사용 불가  
""",
)
@PostMapping("/api/foos")
fun create(...): ApiResponse<String>
```

| 속성 | 필수 | 설명 |
|---|---|---|
| `summary` | ✅ | 한 줄 요약 — `{도메인} {동작}` 형식 |
| `description` | ❌ | 제약 조건, 비즈니스 규칙 — `"""  """` 사용, 줄 끝 공백 2칸으로 개행 |

---

## @Parameter — PathVariable / QueryParam 설명

```kotlin
@Operation(summary = "부품 단건 조회")
@GetMapping("/api/foos/{id}")
fun getById(
    @Parameter(description = "부품 ID", example = "1") @PathVariable id: Long,
): ApiResponse<FetchFooDTO>
```

---

## @Schema — Request DTO 필드 설명

`@Schema` 는 Request DTO 에만 선언. application model 오염 금지.

```kotlin
data class CreateFooRequest(
    @field:Schema(description = "부품명", example = "SUS304", required = true)
    @field:NotBlank
    val name: String,

    @field:Schema(description = "부품 코드", example = "ITEM-001", required = true)
    @field:NotBlank
    val code: String,

    @field:Schema(description = "설명", example = "스테인리스 소재")
    val description: String? = null,
)
```

| 속성 | 설명 |
|---|---|
| `description` | 필드 설명 |
| `example` | 예시값 — Swagger UI 에서 Try it out 시 자동 입력 |
| `required` | 필수 여부 (`true` / `false`) |
| `defaultValue` | 기본값 표시 |
| `allowableValues` | 허용값 목록 (enum 대체) |

---

## @ApiResponses — 응답 코드 명시 (선택)

에러 케이스가 명확하거나 클라이언트에게 중요한 엔드포인트에만 작성.

```kotlin
@Operation(summary = "부품 생성")
@ApiResponses(
    ApiResponse(responseCode = "201", description = "생성 성공"),
    ApiResponse(responseCode = "400", description = "유효성 검사 실패"),
    ApiResponse(responseCode = "409", description = "코드 중복"),
)
@PostMapping("/api/foos")
fun create(...): ApiResponse<String>
```

### 공통 상태코드

| 코드 | 설명 | 주로 사용 |
|---|---|---|
| `200` | 성공 | 조회, 수정, 삭제 |
| `201` | 생성 성공 | 등록 |
| `400` | 유효성 검사 실패 | `@Valid` 실패 |
| `401` | 인증 실패 | 토큰 없음/만료 |
| `403` | 권한 없음 | 접근 권한 부족 |
| `404` | 리소스 없음 | 존재하지 않는 ID |
| `409` | 충돌 | 중복 코드/데이터 |
| `500` | 서버 오류 | 예상치 못한 에러 |

---

## 완성 예시

```kotlin
@RestController
@Tag(name = "001-005. 부품 API", description = "001-005. 부품 API")
class FooCommandApi(
    private val createUsecase: CreateFooUsecase,
) {
    @Operation(
        summary = "부품 생성",
        description = """
신규 부품을 생성합니다.  
- 부품 코드는 중복 불가  
""",
    )
    @ApiResponses(
        ApiResponse(responseCode = "201", description = "생성 성공"),
        ApiResponse(responseCode = "409", description = "코드 중복"),
    )
    @PostMapping("/api/foos")
    fun create(
        @Valid @RequestBody request: CreateFooRequest,
    ): ApiResponse<String> {
        createUsecase.execute(request.toModel())

        return ApiResponseCreated("부품 생성에 성공했습니다.")
    }
}
```

---

## 규칙

- `@Tag` 는 `@RestController` 아래에 선언
- `@Operation` 은 Mapping 어노테이션 위에 선언
- `@Operation(summary = "...")` 필수
- `@Schema` 는 Request DTO 필드에만 — application model 선언 금지
- `@ApiResponses` 는 에러 케이스가 중요한 엔드포인트에만 선언
