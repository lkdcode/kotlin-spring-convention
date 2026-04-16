# adapter/input/web/command — *FormController.kt (SSR)

거의 사용하지 않음. SSR 폼 제출 처리 전용.

```kotlin
@Controller
class FooFormController(
    private val createUsecase: CreateFooUsecase,
    private val deleteUsecase: DeleteFooUsecase,
) {
    @PostMapping("/foos")
    fun create(
        @Valid @ModelAttribute request: CreateFooRequest,
        bindingResult: BindingResult,
        redirectAttributes: RedirectAttributes,
    ): String {
        if (bindingResult.hasErrors()) return "foos/form"

        createUsecase.execute(request.toModel())
        redirectAttributes.addFlashAttribute("message", "생성되었습니다.")

        return "redirect:/foos"
    }

    @PostMapping("/foos/{id}/delete")
    fun delete(
        @PathVariable id: Long,
    ): String {
        deleteUsecase.execute(listOf(id))

        return "redirect:/foos"
    }
}
```

## 규칙

- `@Controller` 사용 — `@RestController` 금지
- 성공 시 `redirect:` 로 리다이렉트 (PRG 패턴)
- 유효성 실패 시 폼 뷰 반환
- HTML form 은 `POST` 만 지원 — `DELETE`/`PUT` 은 `POST` + hidden field 로 처리
- UseCase 만 호출 — 비즈니스 로직 금지
