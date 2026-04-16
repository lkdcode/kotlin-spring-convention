# adapter/input/web/query — *PageController.kt (SSR)

거의 사용하지 않음. SSR 페이지 렌더링 전용.

```kotlin
@Controller
class FooPageController(
    private val fetchUsecase: FetchFooUsecase,
) {
    @GetMapping("/foos")
    fun list(
        model: Model,
        @ModelAttribute queryString: FooQueryString,
        @PageableDefault(size = 15) pageable: Pageable,
    ): String {
        model.addAttribute("data", fetchUsecase.fetchList(queryString.toCondition(), pageable))

        return "foos/list"   // templates/foos/list.html
    }

    @GetMapping("/foos/{id}")
    fun detail(
        @PathVariable id: Long,
        model: Model,
    ): String {
        model.addAttribute("foo", fetchUsecase.getById(FooId(id)))

        return "foos/detail"
    }
}
```

## 규칙

- `@Controller` 사용 — `@RestController` 금지
- 반환값: 템플릿 경로 문자열 (`"foos/list"`)
- `Model` 에 데이터 추가 후 뷰 반환
- UseCase 만 호출 — 비즈니스 로직 금지
