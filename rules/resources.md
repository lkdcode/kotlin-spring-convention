# rules/resources — src/main/resources 구조

## 전체 구조

```
src/main/resources/
├── application.yml              ← 공통 설정 (프로파일 무관)
├── application-local.yml        ← local 환경
├── application-dev.yml          ← dev 환경
├── application-prod.yml         ← prod 환경
│
├── local/                       ← local 전용 설정 파일
│   └── storage/storage-config.yml
│
├── dev/                         ← dev 전용 설정 파일
│   ├── secret/
│   │   ├── jwt/jwt-config.yml
│   │   ├── aws/aws-config.yml
│   │   ├── ncp/ncp-config.yml
│   │   └── slack/slack-config.yml
│   ├── swagger/swagger.yml
│   └── log/log-config.yml
│
├── prod/                        ← prod 전용 설정 파일
│   ├── secret/
│   │   ├── jwt/jwt-config.yml
│   │   └── ...
│   └── log/log-config.yml
│
├── db/                          ← DB 관련
│   ├── migration/               ← Flyway 마이그레이션
│   ├── insert/                  ← 초기 데이터 (환경별)
│   │   ├── local/
│   │   ├── dev/
│   │   └── prod/
│   ├── index/                   ← 인덱스 생성 (참고용)
│   ├── procedure/               ← 저장 프로시저 (필요 시)
│   └── default_column.md        ← 컬럼 공통 규약 문서
│
├── logback-spring.xml           ← 로깅 설정
└── static/                      ← 정적 파일 (이메일 HTML 템플릿 등) — 네이밍: {용도}-{설명}.html
```

---

## application.yml (공통)

모든 환경에서 공통으로 적용. 민감 정보 없음.

```yaml
server:
  port: 8080

spring:
  threads:
    virtual:
      enabled: true

  profiles:
    active: local    # 기본 프로파일

  jpa:
    open-in-view: false

  flyway:
    enabled: true
    locations: classpath:db/migration

  servlet:
    multipart:
      max-file-size: -1
      max-request-size: -1

logging:
  level:
    root: INFO
```

---

## application-{profile}.yml (환경별)

프로파일별 오버라이드. `spring.config.import` 로 세부 설정 파일 주입.

```yaml
# application-local.yml
spring:
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/mydb
    username: root
    password: password

  jpa:
    show-sql: true
    hibernate:
      ddl-auto: validate

  config:
    import:
      - classpath:/local/storage/storage-config.yml   ← local 전용
      - classpath:/dev/secret/jwt/jwt-config.yml      ← 민감 정보
      - classpath:/dev/swagger/swagger.yml
      - classpath:/dev/log/log-config.yml
```

```yaml
# application-prod.yml
spring:
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://prod-host:3306/mydb
    username: ${DB_USERNAME}     ← 환경변수 권장
    password: ${DB_PASSWORD}

  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

  config:
    import:
      - classpath:/prod/secret/jwt/jwt-config.yml
      - classpath:/prod/log/log-config.yml
```

---

## {profile}/secret/ — 민감 정보

서비스별로 분리. 환경별로 값이 다름. **git 에 커밋하지 않는 것 권장.**

```
{profile}/secret/
├── jwt/jwt-config.yml
├── aws/aws-config.yml
├── ncp/ncp-config.yml
├── slack/slack-config.yml
└── ...
```

```yaml
# dev/secret/jwt/jwt-config.yml
jwt:
  secret-key: {사용자 정의}
  issuer: {사용자 정의}
  access:
    expired: {사용자 정의}
  refresh:
    expired: {사용자 정의}
```

```yaml
# dev/secret/aws/aws-config.yml
aws:
  app:
    access_key_id: {사용자 정의}
    secret_access_key: {사용자 정의}
    bucket_name: {사용자 정의}
    region: ap-northeast-2
  presigned:
    get_expired: {사용자 정의}
    put_expired: {사용자 정의}
```

---

## db/migration/ — Flyway 마이그레이션

### 파일 네이밍

버전 번호는 **3자리** 로 통일 — 파일 탐색기 정렬 일관성.

```
V{NNN}__{ACTION}_{TABLE}.sql

V001__INIT_users.sql
V002__INIT_users_groups.sql
V019__ALTER_users_add_email.sql
V020__IDX_users_login_history.sql
```

| prefix | 의미 | 예시 |
|---|---|---|
| `V{NNN}__INIT_` | 테이블 최초 생성 | `V001__INIT_users.sql` |
| `V{NNN}__ALTER_` | 컬럼 추가/변경 | `V010__ALTER_users_add_phone.sql` |
| `V{NNN}__IDX_` | 인덱스 생성 | `V015__IDX_users_login_history.sql` |
| `V{NNN}__INSERT_` | 기준 데이터 삽입 | `V005__INSERT_system_codes.sql` |
| `V{NNN}__DROP_` | 컬럼/테이블 삭제 | `V020__DROP_is_read_from_invitations.sql` |

### 개발 워크플로우

#### Local DB 로 개발 (flywayCleanLocal)

로컬 DB 에 개발할 때. INIT 파일 수정 + Clean 으로 Migration 파일 최소화.

```
컬럼 추가 필요 (로컬 개발 중)
    ↓
V001__INIT_foo.sql 직접 수정 (ALTER 파일 추가 X)
    ↓
./gradlew flywayCleanLocal    ← clean → migrate 재실행
    ↓
테이블 처음부터 재생성 → Migration 1개로 유지

DEV 반영 시점
    ↓
V{N+1}__ALTER_foo_add_column.sql 추가 (이때만 ALTER)
```

```kotlin
// build.gradle.kts
tasks.register("flywayCleanLocal") {
    group = "flyway"
    description = "local DB clean + migrate"
    doLast {
        org.flywaydb.core.Flyway.configure()
            .dataSource(localDbUrl, dbUser, dbPassword)
            .locations("filesystem:src/main/resources/db/migration")
            .cleanDisabled(false)
            .load()
            .also { it.clean(); it.migrate() }
    }
}
```

---

#### DEV DB 에 직접 연결해서 개발

`application-local.yml` 의 datasource 를 DEV DB 로 설정하면, Flyway 가 `flyway_schema_history` 를 읽어 **이미 적용된 migration 은 건너뛰고 새 파일만 자동 실행**.

```
local 에서 DEV DB 연결
    ↓
./gradlew flywayMigrate
    ↓
Flyway 가 flyway_schema_history 확인
    ↓
V001 ~ V005 이미 적용됨 → 스킵
V006__INIT_new_table.sql 미적용 → 실행
    ↓
jOOQ 코드 생성 (현재 DEV 스키마 기준)
```

DEV DB 에 연결하면 별도 로컬 DB 없이도 개발 가능. 단, DEV 데이터에 영향을 줄 수 있으므로 주의.

---

#### DEV Clean (요주의)

DEV DB clean 은 **모든 데이터가 삭제**되므로 반드시 팀 검토 후 진행.

```kotlin
tasks.register("flywayCleanDev") {
    group = "flyway"
    description = "⚠️ DEV DB clean + migrate — 팀 검토 필요"
    doLast {
        // 실행 전 확인 프롬프트
        print("⚠️ DEV DB 의 모든 데이터가 삭제됩니다. 계속하시겠습니까? (yes/no): ")
        val input = readLine()
        if (input?.lowercase() != "yes") {
            println("취소되었습니다.")
            return@doLast
        }

        org.flywaydb.core.Flyway.configure()
            .dataSource(devDbUrl, dbUser, dbPassword)
            .locations("filesystem:src/main/resources/db/migration")
            .cleanDisabled(false)
            .load()
            .also { it.clean(); it.migrate() }
    }
}
```

### SQL 작성 포맷

컬럼명 / 타입 / 제약 / DEFAULT / COMMENT 를 세로 정렬.

**MySQL**

```sql
CREATE TABLE foos
(
    `id`               BIGINT UNSIGNED    NOT NULL  AUTO_INCREMENT PRIMARY KEY  COMMENT '식별 인덱스',

    `name`             VARCHAR(100)       NOT NULL                              COMMENT '이름',
    `code`             VARCHAR(20)        NOT NULL                              COMMENT '코드',
    `description`      VARCHAR(200)       NULL                                  COMMENT '설명',

    `created_at`       DATETIME           NOT NULL  DEFAULT CURRENT_TIMESTAMP   COMMENT '데이터가 생성된 시간',
    `updated_at`       DATETIME           NOT NULL  DEFAULT CURRENT_TIMESTAMP   COMMENT '데이터가 수정된 시간',
    `created_by`       BIGINT UNSIGNED    NOT NULL  DEFAULT 0                   COMMENT '데이터를 생성한 유저 아이디',
    `updated_by`       BIGINT UNSIGNED    NOT NULL  DEFAULT 0                   COMMENT '데이터를 수정한 유저 아이디',
    `is_deleted`       TINYINT(1)         NOT NULL  DEFAULT 0                   COMMENT '소프트 삭제 여부 (`0`: 정상, `1`: 삭제)',
    `deleted_at`       DATETIME           NULL                                  COMMENT '소프트 삭제된 시간',
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**PostgreSQL**

```sql
CREATE TABLE foos
(
  id              BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,

  name            VARCHAR(100)     NOT NULL,
  code            VARCHAR(20)      NOT NULL,
  description     VARCHAR(200)     NULL,

  created_at      TIMESTAMPTZ      NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ      NOT NULL DEFAULT NOW(),
  created_by      BIGINT           NOT NULL DEFAULT 0,
  updated_by      BIGINT           NOT NULL DEFAULT 0,
  is_deleted      BOOLEAN          NOT NULL DEFAULT FALSE,
  deleted_at      TIMESTAMPTZ      NULL,
);

COMMENT ON COLUMN foos.name            IS '이름';
COMMENT ON COLUMN foos.code            IS '코드';
COMMENT ON COLUMN foos.description     IS '설명';
COMMENT ON COLUMN foos.created_at      IS '데이터가 생성된 시간';
COMMENT ON COLUMN foos.updated_at      IS '데이터가 수정된 시간';
COMMENT ON COLUMN foos.created_by      IS '데이터를 생성한 유저 아이디';
COMMENT ON COLUMN foos.updated_by      IS '데이터를 수정한 유저 아이디';
COMMENT ON COLUMN foos.is_deleted      IS '소프트 삭제 여부';
COMMENT ON COLUMN foos.deleted_at      IS '소프트 삭제된 시간';
```

**정렬 규칙**

| 항목 | 규칙 |
|---|---|
| 컬럼명 | 왼쪽 정렬, 가장 긴 컬럼명 기준으로 공백 패딩 |
| 타입 | 컬럼명 뒤 공백으로 세로 정렬 |
| NOT NULL / NULL | 타입 뒤 공백으로 세로 정렬 |
| DEFAULT | NOT NULL 뒤 공백으로 세로 정렬 |
| COMMENT (MySQL) | DEFAULT 뒤 공백으로 세로 정렬 |
| COMMENT (PostgreSQL) | 테이블 생성 후 별도 `COMMENT ON COLUMN` 으로 세로 정렬 |
| id | 첫 번째, 개행으로 분리 |
| BaseEntity 컬럼 | 비즈니스 컬럼 아래, 개행으로 분리 |

---

### 규칙

- 버전 번호 **3자리** 고정 — `V001`, `V010`, `V100`
- 버전 번호는 순차 증가, 절대 변경/삭제 금지
- **로컬 개발**: INIT 파일 직접 수정 + `flywayCleanLocal` — ALTER 파일 축적 방지
- **DEV DB 직접 연결 개발**: `flyway_schema_history` 기반 자동 진행 — 새 파일만 실행
- **DEV Clean**: 팀 검토 후 진행, 확인 프롬프트 필수
- **PROD Clean**: 절대 금지
- 한 번 DEV 에 반영된 파일 내용 수정 금지 — 새 버전으로 추가

---

## db/insert/ — 개발용 벌크 데이터

Flyway migration 과 **별개**. 수동으로 실행하는 개발용 시드 데이터 스크립트.
`flywayCleanLocal` 이후 개발 환경을 셋업할 때 실행.

```
db/insert/
├── local/   ← 로컬 개발 테스트 데이터
├── dev/     ← dev 서버 초기 데이터
└── prod/    ← prod 최소 필수 데이터
```

### 파일 네이밍

버전 번호 **3자리** 고정.

```
I{NNN}__{TABLE}.sql

I001__users_groups.sql
I002__users.sql
I003__system_codes.sql
```

### 사용 흐름

```
flywayCleanLocal 실행 (스키마 초기화)
    ↓
DB 툴 또는 스크립트로 insert 파일 수동 실행
    ↓
개발 준비 완료
```

### 규칙

- Flyway 가 자동 실행하지 않음 — **수동 실행**
- 개발/테스트 목적 데이터만 — prod 실서비스 데이터 포함 금지
- `flywayCleanLocal` 이후 실행 가능 — clean 으로 데이터가 초기화되므로 재삽입 필요 시 활용

---

## {profile}/swagger/swagger.yml

Swagger UI. **dev 에서만 노출**, path 설정이 핵심.

```yaml
# dev/swagger/swagger.yml
springdoc:
  swagger-ui:
    path: /dev/docs          ← 필수
    operations-sorter: alpha
    tags-sorter: alpha

  # group-configs 는 프로젝트별 선택사항 (API 그룹 분류)
  group-configs:
    - group: 전체 API
      paths-to-match: /**
```

---

## {profile}/log/log-config.yml + logback-spring.xml

### log-config.yml — 로그 경로 설정

```yaml
# dev/log/log-config.yml
logging:
  file:
    path: /var/log/app     # 로그 파일 저장 경로 (운영 환경)
```

```yaml
# prod/log/log-config.yml
logging:
  file:
    path: /app/log
```

### logback-spring.xml

`<springProfile>` 로 환경별 appender 분리. 루트에 단일 파일로 관리.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- log-config.yml 에서 주입 -->
    <springProperty name="LOG_PATH" source="logging.file.path" defaultValue="./logs"/>

    <!-- 공통 패턴 -->
    <property name="LOG_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"/>

    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- File Appender (prod 전용) — 일별 롤링 -->
    <appender name="DAILY" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/%d{yyyy,aux}/%d{MM,aux}/%d{yyyy-MM-dd}_app.log</fileNamePattern>
            <maxHistory>90</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>

    <!-- local: 콘솔만 -->
    <springProfile name="local">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- dev: 콘솔만 -->
    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- prod: 파일 + 콘솔 -->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="DAILY"/>
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

</configuration>
```

### 환경별 동작

| | local | dev | prod |
|---|---|---|---|
| Console | ✅ | ✅ | ✅ |
| File | ❌ | ❌ | ✅ (일별 롤링) |
| 보관 기간 | — | — | 90일 |

---

## Dockerfile

```dockerfile
FROM eclipse-temurin:21-jdk-alpine
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

JVM 21 (Virtual Thread 지원) + Alpine 경량 이미지. JVM 플래그 추가 시:

```dockerfile
ENTRYPOINT ["java", "-Djdk.tracePinnedThreads=full", "-jar", "/app.jar"]
```

---

## 규칙

- `application.yml` — 민감 정보 없음, 공통 설정만
- 민감 정보 (`secret/`) — `spring.config.import` 로 주입
- Flyway migration 파일 — 한 번 실행 후 수정 절대 금지
- 환경별 설정 분리 — local/dev/prod 디렉토리로 명확히 구분
- `logback-spring.xml` — 루트에 위치, 프로파일별 다른 설정은 내부 `<springProfile>` 활용
