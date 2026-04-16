# kotlin-spring-convention

Kotlin + Spring Boot 백엔드 프로젝트를 위한 컨벤션 가이드.

헥사고날 아키텍처(Ports & Adapters) 기반으로, 도메인 설계부터 인프라 구성까지 전 계층의 코드 작성 규칙을 정의한다.

## Stack

| 항목 | 기술 |
|---|---|
| Language | Kotlin |
| Runtime | JVM 21 (LTS), Virtual Thread |
| Framework | Spring Boot 3.x (WebMVC) |
| ORM / SQL | JPA + jOOQ DSLContext |
| DB | MySQL, PostgreSQL, MongoDB |
| Cache | Caffeine (Local), Redis (Remote) |
| Messaging | Kafka, Spring Event |
| Security | JWT (Stateless) |
| Storage | Local, AWS S3 (Pre-signed URL) |
| Cloud | AWS, NCP |
| Infra | Redis, Kafka, Slack |
| Build | Gradle (buildSrc, *Libs.kt) |
| Migration | Flyway |

## Structure

```
kotlin-spring-convention/
├── SKILL.md                ← Claude Code 스킬 진입점
├── stack.md                ← 기술 스택, Virtual Thread, JPA vs jOOQ
│
├── domains/                ← 헥사고날 아키텍처 계층별 컨벤션
│   ├── domain/             ← Aggregate, Entity, VO, Spec, Policy, Event, Exception
│   ├── application/        ← UseCase, Service, Input/Output Ports
│   └── adapter/            ← REST, Web, Kafka, Schedule, JPA, jOOQ, MongoDB
│
├── framework/              ← 횡단 관심사
│   ├── api/                ← ApiResponse, ApiException, ApiAdvice, ApiWebClient
│   ├── security/           ← JWT, Filter, Auth, Handler, AOP
│   ├── cache/              ← Caffeine (Local), Redis (Remote)
│   ├── storage/            ← Direct / Pre-signed Upload (Local, S3)
│   ├── jpa/                ← BaseEntity, Validatable
│   └── util/               ← TimeUtil, throwIf, throwUnless
│
├── infra/                  ← 인프라 서비스 (Redis, Kafka, Slack, MongoDB)
├── cloud/                  ← 클라우드 벤더 (AWS S3, NCP SMS)
│
└── rules/                  ← 계층 무관 공통 규칙
    ├── db.md               ← RDB DDL 네이밍 (테이블, 컬럼, 인덱스)
    ├── mongodb.md          ← MongoDB 네이밍 (컬렉션, 필드, 인덱스)
    ├── code-style.md       ← Kotlin 코드 스타일
    ├── dependency.md       ← 의존 방향 규칙
    ├── build.md            ← Gradle buildSrc, *Libs.kt
    ├── resources.md        ← resources 구조, Flyway, Seed
    ├── change-scope.md     ← 커밋 범위, fix/refactor/feat 분리
    └── gitignore.md        ← .gitignore 템플릿
```

## Claude Code Skill

이 저장소는 [Claude Code](https://claude.com/claude-code) 스킬로 사용할 수 있다.

```bash
# clone
git clone git@github.com:{owner}/kotlin-spring-convention.git ~/kotlin-spring-convention

# symlink 등록
ln -s ~/kotlin-spring-convention ~/.claude/skills/kotlin-spring-convention
```

등록 후 Claude Code 세션에서 Kotlin Spring 백엔드 작업 시 자동으로 컨벤션이 적용된다.

직접 호출: `/kotlin-spring-convention`