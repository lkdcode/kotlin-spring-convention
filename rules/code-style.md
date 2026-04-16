# rules/code-style — Kotlin 코드 작성 규칙

계층 무관하게 모든 코드에 적용.

---

## 1. 파라미터 개행

파라미터는 개수에 상관없이 **무조건 개행**.

```kotlin
// ✅
data class FooQueryString(
    val name: String? = null,
    val code: String? = null,
)

// ✅ 파라미터 1개도 개행
fun fetchList(
    condition: FooSearchCondition,
): FetchFooDTOList

// ❌
data class FooQueryString(val name: String? = null, val code: String? = null)
fun fetchList(condition: FooSearchCondition): FetchFooDTOList
```

---

## 2. Trailing Comma

파라미터/인자 마지막에 `,` 필수. 새 항목 추가 시 diff 를 깔끔하게 유지.

```kotlin
// ✅
data class CreateFooModel(
    val name: String,
    val code: String,       // ← trailing comma
)

// ❌
data class CreateFooModel(
    val name: String,
    val code: String
)
```

함수 호출, 컬렉션에도 동일하게 적용.

```kotlin
// ✅
ParamConditionList.of(
    FooConditionKey.NAME to name,
    FooConditionKey.CODE to code,   // ← trailing comma
)

listOf(
    "a",
    "b",    // ← trailing comma
)
```

---

## 3. Expression Body

단일 표현식은 `=` 사용. 블록 `{}` 불필요.

```kotlin
// ✅
fun fetchLookupList(): List<FooLookupDTO> =
    queryPort.fetchLookupList()

override fun execute(model: CreateFooModel) =
    commandPort.create(model)

// ❌
fun fetchLookupList(): List<FooLookupDTO> {
    return queryPort.fetchLookupList()
}
```

2줄 이상이거나 중간 로직이 있으면 블록 사용.

```kotlin
// ✅ 중간 로직 있으면 블록
fun create(request: CreateFooRequest): ApiResponse<String> {
    createUsecase.execute(request.toModel())

    return ApiResponseCreated("생성 성공")
}
```

---

## 4. 어노테이션 순서

**해당 클래스/메서드에 더 관련있는 것이 위, 부가적인 것이 아래.**

```kotlin
// ✅ 클래스 — @RestController 가 본질, @Tag 는 부가(Swagger)
@RestController
@Tag(name = "Foo API")
class FooCommandApi(...)

// ✅ 메서드 — @Operation 이 목적 표현, @PostMapping 은 기술적 바인딩
@Operation(summary = "Foo 생성")
@PostMapping("/api/foos")
fun create(...)

// ✅ Service — @Service 가 본질, @Transactional 은 부가
@Service
@Transactional
class FooCommandService(...)
```

---

## 5. Modifier 순서

`접근제어자 → override → 기타 modifier → fun/val/var`

```kotlin
// ✅
private val name: String
override fun execute(model: CreateFooModel)
private override fun validate()   // ❌ — override 가 앞

// Kotlin 권장 순서
// public / protected / private / internal
// → override
// → abstract / open / final / sealed
// → val / var / fun
```

---

## 6. interface 선언

```kotlin
// ✅ 메서드 2개 이상 개행
interface FooCommandPort {
    fun create(model: CreateFooModel)
    fun update(id: Long, model: UpdateFooModel)
    fun delete(ids: List<Long>)
}

// ✅ 메서드 1개 — 개행 유지 (일관성)
interface CreateFooUsecase {
    fun execute(model: CreateFooModel)
}
```

---

## 7. class / data class 선언

```kotlin
// ✅ 생성자 파라미터 개행
@Service
class FooCommandService(
    private val commandPort: FooCommandPort,
    private val validator: FooValidator,
) : FooCommandUsecase {

    override fun execute(model: CreateFooModel) {
        validator.validateDuplicate(model.code)
        commandPort.create(model)
    }
}

// ✅ data class — init 블록은 본문 가장 위
data class CreateFooModel(
    val name: String,
    val code: String,
) {
    init {
        require(name.isNotBlank()) { "이름은 필수입니다" }
    }
}
```

---

## 8. Lambda / 고차함수

람다가 마지막 파라미터면 `()` 밖으로.

```kotlin
// ✅
listOfNotNull(
    name?.let { FooConditionKey.NAME to it },
    code?.let { FooConditionKey.CODE to it },
)

// ✅ 블록 람다
list.forEach {
    println(it)
}

// ❌
list.forEach({ println(it) })
```

---

## 9. 빈 줄 규칙

- 클래스 내 메서드 사이: 빈 줄 1개
- `execute()` 등 UseCase 호출 후 `return` 전: 빈 줄 1개
- 클래스 본문 시작/끝: 빈 줄 없음

```kotlin
// ✅
class FooCommandService(...) : FooCommandUsecase {

    override fun create(model: CreateFooModel) {
        validator.validateDuplicate(model.code)
        commandPort.create(model)
    }

    override fun delete(ids: List<Long>) {
        commandPort.delete(ids)
    }
}

// ✅ return 전 빈 줄
fun create(request: CreateFooRequest): ApiResponse<String> {
    createUsecase.execute(request.toModel())

    return ApiResponseCreated("생성 성공")
}
```
