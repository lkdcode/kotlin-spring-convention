# adapter/output/command — *CommandAdapter.kt

## 단순 쓰기 — JPA

```kotlin
@Service
class FooCommandAdapter(
    private val repository: FooJpaRepository,
    private val mapper: FooCommandMapper,
) : FooCommandPort {

    override fun save(model: CreateFooModel) =
        mapper.convert(model).let { repository.save(it) }

    override fun update(targetId: Long, model: UpdateFooModel) =
        repository.loadById(targetId).update(model)

    override fun delete(list: List<Long>) =
        repository.findAllById(list).forEach { it.delete() }
}
```

## 복잡한 쓰기 — jOOQ

```kotlin
@Service
class FooCommandAdapter(
    private val repository: FooJpaRepository,
    private val mapper: FooCommandMapper,
    private val dsl: DSLContext,
) : FooCommandPort {

    // 단순 저장 — JPA + Mapper
    override fun save(model: CreateFooModel) =
        mapper.convert(model).let { repository.save(it) }

    // 벌크 UPDATE — jOOQ
    override fun bulkUpdateStatus(ids: List<Long>, status: String) {
        dsl.update(FOO)
            .set(FOO.STATUS, status)
            .set(FOO.UPDATED_AT, LocalDateTime.now())
            .where(FOO.ID.`in`(ids))
            .execute()
    }

    // 조건부 벌크 DELETE — jOOQ
    override fun deleteByCondition(createdBefore: LocalDateTime) {
        dsl.deleteFrom(FOO)
            .where(FOO.CREATED_AT.lt(createdBefore))
            .and(FOO.STATUS.eq("INACTIVE"))
            .execute()
    }

    companion object {
        val FOO = Tables.MST_FOO
    }
}
```

## mapper/*CommandMapper.kt

```kotlin
@Service
class FooCommandMapper {
    fun convert(model: CreateFooModel): FooJpaEntity =
        FooJpaEntity(
            name = model.name,
            code = model.code,
            description = model.description,
        )
}
```

## 규칙
- 단순 저장/수정/삭제 → JPA (`repository.save`, `loadById().update()`)
- 벌크 연산, 복잡한 조건 → jOOQ `DSLContext`
- JPA 와 jOOQ 혼용 시 같은 Adapter 클래스 안에서 목적별로 분리
- jOOQ 테이블 상수는 `companion object` 에 선언
- **Model → JPA Entity 변환은 반드시 `*CommandMapper` 를 통해서** — 직접 생성 금지

## Mapper 규칙
- 위치: `adapter/output/command/mapper/*CommandMapper.kt`
- 클래스명: `*CommandMapper` — 예: `FooCommandMapper`
- 어노테이션: `@Service`
- 역할: application model → JPA Entity 변환만 — 비즈니스 로직 금지
