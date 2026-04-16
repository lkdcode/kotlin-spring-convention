# application/ports/input/command — *Usecase.kt (interface)

## 1개의 command — 메서드명 `.execute()`

클래스명이 의도를 설명하므로 메서드명은 `execute` 로 고정.

```kotlin
interface CreateFooUsecase {
    fun execute(model: CreateFooModel)
}

interface UpdateFooUsecase {
    fun execute(id: Long, model: UpdateFooModel)
}

interface DeleteFooUsecase {
    fun execute(ids: List<Long>)
}
```

## 2개 이상의 command — 메서드명 명시

클래스명만으로 어떤 동작인지 알 수 없으므로 메서드명으로 구분.

```kotlin
interface FooCommandUsecase {
    fun create(model: CreateFooModel)
    fun update(id: Long, model: UpdateFooModel)
    fun delete(ids: List<Long>)
}
```

## 규칙

- 인터페이스 선언만
- command 1개 → 메서드명 `execute`
- command 2개 이상 → 메서드명 명시 (`create`, `update`, `delete` 등)
- 메서드는 역할당 1개 (단일 책임)

## 구현체 네이밍

| 인터페이스 | 구현체 |
|---|---|
| `FooCommandUsecase` | `FooCommandService` |
| `CreateFooUsecase` | `CreateFooService` |
| `UpdateFooUsecase` | `UpdateFooService` |
| `DeleteFooUsecase` | `DeleteFooService` |

## 반환 타입

- 필요 시 도메인 ID, 도메인 객체, application DTO 반환 가능

## Command Model 위치

`application/ports/input/command/model/` — 상세 규칙은 `model/model.md` 참조
