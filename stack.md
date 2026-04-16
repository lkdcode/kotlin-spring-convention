# Tech Stack

| 항목 | 스펙 |
|---|---|
| Language | Kotlin |
| Runtime | JVM 21 (LTS) |
| Framework | Spring Boot 3.x (WebMVC) |
| Thread Model | Virtual Thread (Project Loom) |
| DB | HikariCP |
| ORM (Command) | JPA — 단순 쓰기 |
| SQL (Command) | jOOQ — 복잡한 쓰기 |
| SQL (Query) | jOOQ QueryBuilder — 복잡한 읽기 / JPA 확장함수 — 단순 단건 조회 |

---

## Virtual Thread 활성화

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

---

## Virtual Thread 특성

**장점**
- Blocking I/O (DB, 네트워크) 중 carrier thread 반환 → 스레드 자원 낭비 없음
- Thread-per-request 모델 유지하면서 높은 처리량

---

## Carrier Thread Pinning (고정) 문제

Virtual Thread 는 OS 스레드인 **Carrier Thread** 위에 마운트되어 실행됨.
Blocking I/O 발생 시 Carrier Thread 에서 언마운트(반환)되어 다른 VT 가 사용 가능.

그러나 `synchronized` 블록 안에서 Blocking 이 발생하면 **Carrier Thread 가 고정(pin)** 됨 — Carrier Thread 수만큼만 동시 처리 가능해져 성능 저하.

```kotlin
// ❌ synchronized — Blocking 중에도 carrier thread 고정
@Synchronized
fun criticalSection() {
    heavyIO()  // 이 동안 carrier thread 점유
}

// ✅ ReentrantLock — Blocking 중 carrier thread 반환
private val lock = ReentrantLock()
fun criticalSection() {
    lock.withLock { heavyIO() }
}
```

**진단:** JVM 플래그로 pinning 발생 위치 추적 가능
```
-Djdk.tracePinnedThreads=full
```

**주의 대상:** 일부 JDBC 드라이버, 오래된 라이브러리 내부의 `synchronized` 사용

---

## HikariCP 설정 (Virtual Thread 환경)

**Virtual Thread 수와 무관하게 DB 커넥션 수는 제한됨.**
수천 개의 VT 가 동시에 쿼리를 실행하려 해도 커넥션 풀이 병목.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # DB 서버 처리 능력 기준으로 설정 (VT 수 아님)
      minimum-idle: 5
      connection-timeout: 3000     # 커넥션 획득 대기 최대 3초 (기본 30초)
      idle-timeout: 600000         # 10분 미사용 커넥션 반환
      max-lifetime: 1800000        # 30분 후 커넥션 교체
      keepalive-time: 60000        # 1분마다 keepalive (stale 커넥션 방지)
```

**Pool Size 기준**

| 환경 | 권장 `maximum-pool-size` |
|---|---|
| 기존 Thread-per-request | 요청 처리 스레드 수와 유사 |
| Virtual Thread | **DB 서버 CPU 코어 수 × 2~4** 수준 (수백 설정 금지) |

> VT 환경에서 pool-size 를 크게 늘려도 DB 서버가 병목 — 오히려 connection-timeout 을 짧게 해서 빠른 실패가 유리.

---

## JPA vs jOOQ 선택 기준

| | JPA | jOOQ |
|---|---|---|
| Command | 단순 저장/수정/삭제 | 벌크 연산, 복잡한 조건 UPDATE |
| Query | 단순 단건 조회 (확장함수) | 복잡한 조회, 페이징, JOIN, 조건 검색 |
| 장점 | Entity 생명주기 관리, dirty checking | 타입 안전 SQL, 복잡한 JOIN/서브쿼리 |
| 단점 | N+1, 복잡한 쿼리 불편 | Entity 상태 관리 없음 |

```
단순 save/update/delete          → JPA Repository
벌크 UPDATE/DELETE               → jOOQ DSLContext
단순 단건 조회 (ID 조회, exists) → JPA Repository 확장함수
복잡한 조회, 페이징, JOIN        → jOOQ DSLContext
```

---

## jOOQ fetchCount 성능 주의

```kotlin
val totalSize = dsl.fetchCount(query)  // SELECT COUNT(*) FROM (subquery) — 별도 쿼리 실행
val items = query.limit(...).offset(...).fetch()
```

`fetchCount()` 는 **별도 COUNT 쿼리**를 실행하므로 DB 왕복이 2회 발생.
대용량 테이블에서는 COUNT 자체가 느릴 수 있음.

**대안:**

| 상황 | 방법 |
|---|---|
| 일반적인 페이징 | `fetchCount()` — 구현 단순, 대부분 충분 |
| 대용량 + 정확한 전체 건수 필요 | `COUNT(*) OVER()` 윈도우 함수 (단일 쿼리) |
| 무한 스크롤 (전체 건수 불필요) | `fetchCount()` 생략, `hasNext` 방식으로 대체 |

```kotlin
// COUNT(*) OVER() — 단일 쿼리로 데이터와 전체 건수 동시 조회
val result = dsl
    .select(
        FOO.ID, FOO.NAME,
        DSL.count().over().`as`("totalCount"),
    )
    .from(FOO)
    .where(condition)
    .limit(pageable.pageSize)
    .offset(pageable.offset)
    .fetch()

val total = result.firstOrNull()?.get("totalCount", Long::class.java) ?: 0L
```
