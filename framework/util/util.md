# framework/util — 범용 유틸리티

계층 무관하게 어디서든 사용 가능한 경량 유틸 함수.

## 파일 구조

```
framework/util/
├── TimeUtil.kt           ← KST 시간, 포맷 변환
├── ThrowExtensions.kt    ← throwIf, throwUnless
├── BooleanExtensions.kt  ← isNot, isTrue, isFalse
└── NullExtensions.kt     ← ifNull
```

---

## TimeUtil

KST 기준 시간 유틸리티.

```kotlin
object TimeUtil {
    private val KST: ZoneId = ZoneId.of("Asia/Seoul")
    private val UTC: ZoneId = ZoneId.of("UTC")

    private val TIME_FORMAT: DateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
    val NANO_TIME_FORMAT: DateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSSSSS")

    fun nowKST(): LocalDateTime = Instant.now().atZone(KST).toLocalDateTime()
    fun nowKSTTimeFormat(): String = nowKST().format(TIME_FORMAT)
    fun nowKSTNanoTimeFormat(): String = nowKST().format(NANO_TIME_FORMAT)

    fun convertTime(target: String): LocalDateTime =
        LocalDateTime.parse(target, TIME_FORMAT)

    fun convertTimeDateString(target: String): LocalDateTime =
        LocalDate.parse(target).atStartOfDay()

    fun convertFormat(target: LocalDateTime): String =
        target.format(TIME_FORMAT)

    fun convertUtcToKst(target: LocalDateTime?): LocalDateTime? =
        target?.atZone(UTC)
            ?.withZoneSameInstant(KST)
            ?.toLocalDateTime()
}
```

---

## 예외 throw 제어

조건 방향 실수를 방지. 의도를 명확히 표현.

```kotlin
// 조건이 true 면 throw
fun throwIf(condition: Boolean, exception: Throwable) {
    if (condition) throw exception
}

// 조건이 false 면 throw
fun throwUnless(condition: Boolean, exception: Throwable) {
    if (!condition) throw exception
}

// ApiException 오버로딩
fun throwIf(condition: Boolean, apiResponseCode: ApiResponseCode) =
    throwIf(condition, ApiException(apiResponseCode))

fun throwUnless(condition: Boolean, apiResponseCode: ApiResponseCode) =
    throwUnless(condition, ApiException(apiResponseCode))
```

---

## Boolean / Null 흐름 제어

```kotlin
// Boolean 반전 — !condition 실수 방지
fun isNot(condition: Boolean) = !condition

// true 일 때 실행
inline fun Boolean.isTrue(action: () -> Unit) {
    if (this) action()
}

// false 일 때 실행
inline fun Boolean.isFalse(action: () -> Unit) {
    if (!this) action()
}

// null 일 때 실행
inline fun <T> T?.ifNull(action: () -> Unit): T? {
    if (this == null) action()
    return this
}
```

### 사용 예시

```kotlin
// throwIf / throwUnless
throwUnless(isActive, ApiResponseCode.FOO_NOT_ACTIVE)
throwIf(isDuplicate, ApiResponseCode.FOO_DUPLICATE_CODE)

// isNot
throwIf(isNot(isActive), ApiResponseCode.FOO_NOT_ACTIVE)

// isTrue / isFalse
isDeleted.isTrue { throw ApiException(ApiResponseCode.FOO_ALREADY_DELETED) }
isActive.isFalse { throw ApiException(ApiResponseCode.FOO_NOT_ACTIVE) }

// ifNull
user.ifNull { throw ApiException(ApiResponseCode.USER_NOT_FOUND) }
```

---

## 규칙

- `TimeUtil.nowKST()` — JPA BaseEntity 시간 필드, 비즈니스 로직에서 현재 시간 사용 시 통일
- `throwIf` — 조건이 참일 때 throw ("이게 true 면 안 된다")
- `throwUnless` — 조건이 거짓일 때 throw ("이게 true 여야 한다")
- `isNot`, `isTrue`, `isFalse`, `ifNull` — 가독성 향상, 직접 `!`, `== null` 대신 사용
