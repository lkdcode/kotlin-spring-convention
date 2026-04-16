# adapter/output/infrastructure/mongo/doc — MongoDB Document

## 디렉토리 구조

```
adapter/output/infrastructure/mongo/
├── doc/
│   └── *Document.kt         ← @Document 클래스
└── repository/
    └── *DocumentRepository.kt
```

---

## Document 클래스

```kotlin
@Document(collection = "foos")
class FooDocument(
    @Id
    var id: String? = null,

    @Field("version")
    var version: Int = 1,

    @Field("foo_name")
    var fooName: String,

    @Field("bar_id")
    var barId: Long,

    @Field("ts")
    var ts: Long = nowTS(),          // epoch millis — 정렬용

    @Field("payload")
    var payload: String,

    @Field("created_at")
    var createdAt: Instant = nowUTC(),

    @Field("deleted_at")
    var deletedAt: Instant? = null,

    @Field("is_deleted")
    var isDeleted: Boolean = false,
) {
    fun delete() {
        isDeleted = true
        deletedAt = nowUTC()
    }
}
```

## 외부 시스템 연동 Document (읽기 전용)

외부 MongoDB (IoT, 레거시 등) 에서 읽는 경우 `data class` + `val` 사용.
필드명이 외부 시스템 규칙을 따르면 `@Field` 에 그대로 명시.

```kotlin
@Document("external_io_datas")
data class ExternalIoDataDocument(
    @Id
    val id: String,

    @Field("DEVICE_CODE")       // 외부 시스템 UPPER_CASE 그대로
    val deviceCode: String? = null,

    @Field("BEF_RCV_TIME")
    val befRcvTime: String? = null,
) {
    fun getFieldValue(key: String): String? = when (key.uppercase()) {
        "DEVICE_CODE" -> deviceCode
        else -> null
    }
}
```

---

## 규칙

- 클래스명: `*Document` — JPA Entity 와 혼동 방지
- 패키지: `adapter/output/infrastructure/mongo/doc/` — `jpa/entity/` 와 동일 레벨
- 도메인 직접 설계 Document: `class` + `var` (mutable, soft delete 필요)
- 외부 시스템 연동 Document: `data class` + `val` (read-only)
- `@Field` 명시 필수 — 필드명 변경 시 DB 데이터 손실 방지
- `@Id var id: String?` — 신규 저장 시 null, MongoDB 가 자동 생성
- 외부 시스템 필드명 (`UPPER_CASE`)은 그대로 유지 — 임의 변환 금지
- 네이밍 컨벤션 → `rules/mongodb.md` 참조
