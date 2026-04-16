# rules/mongodb — MongoDB 네이밍 컨벤션

---

## 컬렉션 네이밍

RDB 테이블 네이밍과 동일한 원칙. 도메인 우선, snake_case, 복수형.

```
{domain}                      ← 주 도큐먼트
{domain}_{entity}             ← 하위 도큐먼트 / 관계

foos                  ← foo 도메인 주 컬렉션
foos_bars             ← foo-bar 관계
foos_history          ← foo 이력
```

---

## 필드 네이밍

snake_case — `@Field` 명시 필수.

```kotlin
@Field("foo_id")      var fooId: String
@Field("bar_name")    var barName: String
@Field("created_at")  var createdAt: Instant
@Field("is_deleted")  var isDeleted: Boolean
```

### 외부 시스템 연동 예외

외부 시스템이 정한 필드명은 **그대로 유지** — 임의 변환 금지.

```kotlin
@Field("DEVICE_CODE")   val deviceCode: String?
@Field("BEF_RCV_TIME")  val befRcvTime: String?
```

---

## 인덱스 네이밍

RDB 인덱스 네이밍과 동일한 원칙.

```
idx_{collection}_{field(s)}
uq_{collection}_{field}

idx_foos_bar_name
idx_foos_created_at
idx_foos_foo_id_status        ← 복합 인덱스
uq_foos_foo_id
```

---

## 규칙 요약

| 항목 | 규칙 |
|---|---|
| 컬렉션명 | 도메인 우선, 복수형, snake_case |
| 필드명 | snake_case, `@Field` 명시 필수 |
| 외부 시스템 필드 | 원본 그대로 유지 (UPPER_CASE 등) |
| 인덱스 | `idx_{collection}_{field}` |
| 유니크 인덱스 | `uq_{collection}_{field}` |
