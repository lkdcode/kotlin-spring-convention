# adapter/output/infrastructure/jpa/repository — *JpaRepository.kt

```kotlin
interface FooJpaRepository : JpaRepository<FooJpaEntity, Long> {

    // JPA 메서드 쿼리보다 성능이 필요하지만 jOOQ 까지는 불필요한 경우 — native query 사용
    @Query(value = "SELECT MAX(foo_code) FROM foos", nativeQuery = true)
    fun findMaxCode(): String?
}

// 확장함수 + CustomException 패턴 — Long ID
fun FooJpaRepository.loadById(id: Long): FooJpaEntity =
    findById(id).orElseThrow { throw ApiException(ApiResponseCode.FOO_NOT_FOUND) }

// 필요 시 도메인 ID 타입도 허용
fun FooJpaRepository.loadById(id: FooId): FooJpaEntity =
    findById(id.value).orElseThrow { throw ApiException(ApiResponseCode.FOO_NOT_FOUND) }
```

## @Query 사용 기준

| 상황 | 선택 |
|---|---|
| `findById`, `findAll`, `existsById` 등 기본 제공 | JPA 기본 메서드 |
| 단순 집계, 특정 컬럼 조회 (JOIN 없음) | `@Query` native query |
| 복잡한 JOIN, 페이징, 동적 조건 | jOOQ DSLContext |

## 메서드 쿼리 성능 주의

JPA 메서드 쿼리가 의도치 않은 성능 저하를 일으킬 수 있는 케이스.

| 메서드 쿼리 | 문제 | 대안 |
|---|---|---|
| `existsByCode(code)` | `SELECT COUNT(*)` 전체 집계 | `@Query` LIMIT 1 또는 jOOQ `fetchExists()` |
| `findAllByStatus(status)` | 결과 무제한 로딩 | `Pageable` 필수 또는 jOOQ |
| `deleteByCreatedAtBefore(date)` | SELECT 후 건별 DELETE (N+1) | `@Modifying @Query` 벌크 삭제 또는 jOOQ |
| `findByNameContaining(keyword)` | 인덱스 없으면 풀스캔 | jOOQ + 인덱스 확인 |

```kotlin
// ❌ existsByCode — COUNT(*) 전체 집계
fun existsByCode(code: String): Boolean

// ✅ @Query — LIMIT 1 으로 첫 행 발견 시 즉시 중단
@Query(value = "SELECT 1 FROM foos WHERE code = :code LIMIT 1", nativeQuery = true)
fun existsByCode(code: String): Int?

// ✅ jOOQ — fetchExists() 로 EXISTS 서브쿼리 생성
dsl.fetchExists(dsl.selectOne().from(FOO).where(FOO.CODE.eq(code)))

// ❌ deleteBy* — SELECT 후 건별 DELETE
fun deleteByCreatedAtBefore(date: LocalDateTime)

// ✅ @Modifying — 단일 DELETE 쿼리
@Modifying
@Query(value = "DELETE FROM foos WHERE created_at < :date", nativeQuery = true)
fun deleteByCreatedAtBefore(date: LocalDateTime)
```

## 규칙
- `loadById()` 확장함수: `Long` 또는 도메인 ID 타입 모두 허용
- JPA 메서드 쿼리보다 성능이 필요하고 jOOQ 가 과한 경우 → `@Query` native query
- `existsBy*` 메서드 쿼리 사용 금지 — `@Query LIMIT 1` 또는 jOOQ `fetchExists()` 사용
- `findAllBy*` 는 반드시 `Pageable` 과 함께 사용 — 무제한 로딩 금지
- `deleteBy*` 메서드 쿼리 사용 금지 — `@Modifying @Query` 또는 jOOQ 사용
