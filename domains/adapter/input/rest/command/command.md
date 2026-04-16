# adapter/input/rest/command — *CommandApi.kt

```kotlin
@RestController
@Tag(name = "NNN-NNN. {도메인} 관리 API", description = "NNN-NNN. {도메인} 관리 API")
class FooCommandApi(
    private val createUsecase: CreateFooUsecase,
    private val updateUsecase: UpdateFooUsecase,
    private val deleteUsecase: DeleteFooUsecase,
    private val restoreUsecase: RestoreFooUsecase,
) {
    @Operation(summary = "Foo 생성")
    @PostMapping("/api/foos")
    fun create(
        @Valid @RequestBody request: CreateFooRequest,
    ): ApiResponse<String> {
        createUsecase.execute(request.toModel())

        return ApiResponseCreated("Foo 생성에 성공했습니다.")
    }

    @Operation(summary = "Foo 수정")
    @PutMapping("/api/foos/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateFooRequest,
    ): ApiResponse<String> {
        updateUsecase.execute(id, request.toModel())

        return ApiResponseUpdated("Foo 수정에 성공했습니다.")
    }

    @Operation(summary = "Foo 삭제")
    @DeleteMapping("/api/foos")
    fun delete(
        @RequestBody ids: List<Long>,
    ): ApiResponse<String> {
        deleteUsecase.execute(ids)

        return ApiResponseDeleted("Foo 삭제에 성공했습니다.")
    }

    @Operation(summary = "Foo 복원")
    @PatchMapping("/api/foos/{id}/restore")
    fun restore(
        @PathVariable id: Long,
    ): ApiResponse<String> {
        restoreUsecase.execute(id)

        return ApiResponseUpdated("Foo 복원에 성공했습니다.")
    }
}
```

## 규칙

- 클래스명: `*CommandApi`
- UseCase 별로 주입 — `createUsecase`, `updateUsecase`, `deleteUsecase` 등 `*Usecase` 네이밍
- 반환: `ApiResponse<T>` — 상세는 `rest/common/common.md` 참조
- `@RequestBody` 는 Request DTO 로 받고 `toModel()` 로 변환 — application model 직접 수신 금지
- Request / Response DTO, Mapper 규칙: `rest/common/dto.md` 참조
- 메서드명: HTTP 동작 그대로 — `create`, `update`, `delete`, `restore` (`getXxx` 금지)
- `@Valid` 는 `@RequestBody` 에만 — `@PathVariable` 제외
- `@PathVariable` 파라미터명 일치 시 `name` 생략
- `@RequestMapping` 사용 절대 금지 — 각 메서드에 풀 경로 작성
- 엔드포인트 네이밍: `/api` 로 시작 후 도메인 표현
- `execute()` 호출 후 한 줄 개행 후 `return` 작성
- 비즈니스 로직 작성 금지 — UseCase 호출만
- `@Operation(summary = "...")` 필수
