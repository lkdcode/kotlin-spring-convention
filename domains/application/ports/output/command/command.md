# application/ports/output/command — *CommandPort.kt (interface)

## 공통 규칙 (ports/output 전체)

- 타 도메인의 동일 포트와 중복 작성 금지 — 단, 도메인 정책이 다른 경우 분리 고려
- 단일 책임 원칙 준수 — 하나의 액터 기준으로 책임 분리
- 구현체는 `adapter/output/command/` 에 동일 구조로 위치
- 네이밍: `XxxCommandPort`

## CommandPort 규칙

```kotlin
interface FooCommandPort {
    fun create(model: CreateFooModel): FooId   // 생성 후 ID 필요 시
    fun create(model: CreateFooModel): Unit    // ID 불필요 시
    fun createAll(models: List<CreateFooModel>)
    fun update(id: FooId, model: UpdateFooModel)
    fun delete(id: FooId)
    fun deleteAll(ids: List<FooId>)
}
```

## 함수 네이밍

| 작업 | 네이밍 |
|---|---|
| 단건 생성 | `create` |
| 다건 생성 | `createAll` |
| 수정 | `update` |
| 삭제 | `delete`, `deleteAll` |

## 반환 타입

| 반환 타입 | 예시 | 사용 시점 |
|---|---|---|
| `Unit` | — | 반환값 불필요 |
| 도메인 ID | `FooId` | 생성 후 ID만 필요 시 |
| Value Object | `Money`, `FooCode` | 계산/변환 결과값 필요 시 |
| Model (application) | `CreateFooModel` | 생성/수정 결과 모델 필요 시 |

- **JPA Entity 반환 절대 금지** — adapter 계층 유출
- **DTO 반환 금지** — QueryPort 책임
