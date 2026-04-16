# adapter/output/infrastructure/jpa/entity — *Entity.kt

JPA / R2DBC 모두 동일한 컨벤션을 따른다. 어노테이션만 다름.

---

## JPA Entity

```kotlin
@Entity
@Table(name = "foos")
class FooJpaEntity(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null,

    // @Column 에 제약 조건 명시 (unique, nullable, length)
    @Column(name = "foo_code", unique = true, nullable = false, length = 20)
    var fooCode: String,

    @Column(name = "type_id")
    var typeId: Long? = null,

    // jakarta.validation 제약 상수는 domain/spec 에서 참조
    @field:NotBlank(message = FooSpec.REQUIRED_NAME_MESSAGE)
    @field:Size(max = FooSpec.NAME_LENGTH, message = FooSpec.INVALID_NAME_LENGTH_MESSAGE)
    @Column(name = "name", length = 100)
    var name: String,

    @Column(name = "description", length = 100)
    var description: String? = null,


    fun update(model: UpdateFooModel) {
        name = model.name
        description = model.description
        if (model.isDeleted) delete() else restore()
        validate()   // 상태 변경 후 항상 validate
    }

    init {
        validate()   // 생성 시 validate
    }
}
```

## R2DBC Entity

어노테이션 패키지만 다르고 구조는 동일.

```kotlin
@Table("foos_history")
class FooHistoryEntity(
    @Id
    var id: Long? = null,

    @Column("foo_id")
    var fooId: Long,

    @Column("bar_id")
    var barId: Long,

    // 시간 필드는 Instant
    @Column("created_at")
    var createdAt: Instant,
) : BaseEntity()
```

## BaseEntity (JPA)

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity : Validatable {

    @CreatedDate  @Column(name = "created_at", nullable = false)
    var createdAt: LocalDateTime = TimeUtil.nowKST()

    @LastModifiedDate  @Column(name = "updated_at", nullable = false)
    var updatedAt: LocalDateTime = TimeUtil.nowKST()

    @CreatedBy  @Column(name = "created_by", nullable = false)
    var createdBy: Long = 0

    @LastModifiedBy  @Column(name = "updated_by", nullable = false)
    var updatedBy: Long = 0

    @Column(name = "is_deleted", nullable = false)
    private var isDeleted: Boolean = false

    @Column(name = "deleted_at")
    private var deletedAt: LocalDateTime? = null

    fun delete()   { isDeleted = true;  deletedAt = TimeUtil.nowKST() }
    fun restore()  { isDeleted = false }
    fun isDeleted(): Boolean    = isDeleted
    fun isNotDeleted(): Boolean = !isDeleted

    @PrePersist fun onCreate() { createdAt = TimeUtil.nowKST(); updatedAt = TimeUtil.nowKST() }
    @PreUpdate  fun onUpdate() { updatedAt = TimeUtil.nowKST() }
}
```

> `BaseTimeEntity` 등 감사 필드 없이 시간만 필요한 경우 별도 상속 가능.

---

## 규칙

### 어노테이션
- JPA: `jakarta.persistence.*` (`@Entity`, `@Table`, `@Column`, `@Id`, `@GeneratedValue`)
- R2DBC: `org.springframework.data.relational.core.mapping.*` (`@Table`, `@Column`) + `org.springframework.data.annotation.@Id`

### 필드
- `var` : 변경 가능한 도메인 필드, 감사 필드
- `val` : 변경 불가 필드 (생성 후 고정 — 비즈니스 규칙상 불변인 값)
- `var id: Long? = null` : PK는 항상 nullable var

### 시간
- 도메인 시간 필드: `Instant`
- BaseEntity 감사 필드: `LocalDateTime` (TimeUtil.nowKST() 사용)

### @Column 제약
- DB 스키마 제약은 `@Column` 에 명시: `nullable`, `length`, `unique`
- 애플리케이션 유효성 검사는 `@field:` prefix + `domain/spec` 상수 참조

### update() / init
- 상태 변경 메서드 끝에 `validate()` 호출
- `init { validate() }` — 생성 시점에도 검증
- soft delete: `delete()` / `restore()` — BaseEntity 제공, 물리 삭제 금지
