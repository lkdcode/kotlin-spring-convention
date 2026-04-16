# application/ports/input/command/model — Command Model

UseCase 의 입력 계약. HTTP 관심사(`@field:NotBlank` 등) 포함 금지.
생성 시 `init` 블록에서 도메인 규칙 기반 유효성 검사.

```kotlin
data class CreateFooModel(
    val name: String,
    val code: String,
    val description: String? = null,
) {
    init {
        require(name.isNotBlank()) { FooSpec.REQUIRED_NAME_MESSAGE }
        require(code.isNotBlank()) { FooSpec.REQUIRED_CODE_MESSAGE }
    }
}

data class UpdateFooModel(
    val name: String,
    val description: String? = null,
) {
    init {
        require(name.isNotBlank()) { FooSpec.REQUIRED_NAME_MESSAGE }
    }
}
```

## 규칙

- `@field:NotBlank`, `@field:NotNull` 등 Jakarta Validation 어노테이션 금지
- `init` 블록에서 `require()` 로 검증 — 상수는 `domain/spec` 참조
- HTTP 요청 매핑은 `adapter/input/rest/command/request/` 의 Request DTO 가 담당
- `List<XxxModel>`, `List<Long>` 직접 사용 허용
- 유효성 검사나 비즈니스 연산이 필요하면 일급 컬렉션으로 래핑
- **JPA Entity, adapter 계층 타입 사용 금지**
