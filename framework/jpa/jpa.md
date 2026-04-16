# framework/jpa — JPA 기반 클래스

## 구성 요소

| 클래스/인터페이스 | 역할 |
|---|---|
| `Validatable` | 검증 가능한 엔티티 마커 인터페이스 |
| `BaseTimeEntity` | 시간 필드만 (createdAt, updatedAt) |
| `BaseEntity` | 감사 + soft delete 포함 전체 |
| `validate()` | `@field:` 제약을 런타임 검증하는 확장 함수 |

---

## Validatable

```kotlin
interface Validatable
```

`validate()` 확장 함수를 사용하려면 반드시 구현.

---

## ValidatorExtensions

```kotlin
private val validator: Validator by lazy {
    Validation.buildDefaultValidatorFactory().validator
}

fun <T : Validatable> T.validate() {
    val violations = validator.validate(this)
    throwIf(violations.isNotEmpty(), ConstraintViolationException(violations))
}
```

`init { validate() }` 와 `update()` 메서드 끝에서 호출 — `@field:` 어노테이션 기반 제약 검증.

---

## BaseTimeEntity

감사 필드 없이 **시간만** 필요한 경우.

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseTimeEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false)
    var createdAt: LocalDateTime = TimeUtil.nowKST()

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    var updatedAt: LocalDateTime = TimeUtil.nowKST()

    @PrePersist
    fun onCreate() {
        createdAt = TimeUtil.nowKST()
        updatedAt = TimeUtil.nowKST()
    }

    @PreUpdate
    fun onUpdate() {
        updatedAt = TimeUtil.nowKST()
    }
}
```

---

## BaseEntity

감사 필드(`createdBy`, `updatedBy`) + soft delete.

```kotlin
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
abstract class BaseEntity : Validatable {

    @CreatedDate
    @Column(name = "created_at", nullable = false)
    var createdAt: LocalDateTime = TimeUtil.nowKST()

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    var updatedAt: LocalDateTime = TimeUtil.nowKST()

    @CreatedBy
    @Column(name = "created_by", nullable = false)
    var createdBy: Long = 0

    @LastModifiedBy
    @Column(name = "updated_by", nullable = false)
    var updatedBy: Long = 0

    @Column(name = "is_deleted", nullable = false)
    private var isDeleted: Boolean = false

    @Column(name = "deleted_at")
    private var deletedAt: LocalDateTime? = null

    open fun delete() {
        isDeleted = true
        deletedAt = TimeUtil.nowKST()
    }

    fun restore() {
        isDeleted = false
    }

    fun isDeleted(): Boolean = isDeleted
    fun isNotDeleted(): Boolean = !isDeleted

    @PrePersist
    fun onCreate() {
        createdAt = TimeUtil.nowKST()
        updatedAt = TimeUtil.nowKST()
    }

    @PreUpdate
    fun onUpdate() {
        updatedAt = TimeUtil.nowKST()
    }
}
```

---

## 상속 기준

| 상황 | 선택 |
|---|---|
| 생성/수정 시간만 필요 | `BaseTimeEntity` |
| 생성자/수정자 추적 필요 | `BaseEntity` |
| soft delete 필요 | `BaseEntity` |
| 이력성 데이터 (변경 불가) | 두 가지 모두 미상속 |

---

## 규칙

- 물리 삭제 금지 — `delete()` / `restore()` 로 soft delete 처리
- `Validatable` 구현 시 `init { validate() }` 필수 — 생성 시점 검증
- `update()` 메서드 마지막에 `validate()` 호출 — 상태 변경 후 검증
- `delete()` 는 `open` — 하위 엔티티에서 추가 로직 오버라이드 가능
