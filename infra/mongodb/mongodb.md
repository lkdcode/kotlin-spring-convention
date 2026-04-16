# infra/mongodb — MongoDB

Spring Data MongoDB. `@EnableMongoRepositories` 로 패키지 설정.

## 설정

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/mydb
      # 또는 분리 설정
      host: localhost
      port: 27017
      database: mydb
      username: {사용자 정의}
      password: {사용자 정의}
```

## MongoDBConfig

여러 MongoDB 인스턴스 또는 Repository 패키지 분리가 필요할 때 설정.

```kotlin
@Configuration
@EnableMongoRepositories(
    basePackages = ["com.example.domains"],
    mongoTemplateRef = "mongoTemplate",
)
class MongoDBConfig
```

단일 MongoDB 인스턴스면 `@EnableMongoRepositories` 없이 Spring Boot 자동 구성 사용.

## 인덱스 관리

Flyway 없음 — `db/mongodb/` 에 shell 명령어로 관리.

```
src/main/resources/db/mongodb/
└── V001__IDX_{collection}.md
```

```shell
# V001__IDX_foos.md
db.foos.createIndex({ "bar_id": 1, "created_at": -1 })
db.foos.createIndex({ "foo_name": 1 })
```

## 규칙

- 인덱스는 `db/mongodb/` 에 shell 명령어로 문서화 — 직접 실행
- `@EnableMongoRepositories` 는 멀티 MongoDB 또는 패키지 분리 시에만 선언
- Document 패턴: `domains/adapter/output/infrastructure/mongo/doc/doc.md` 참조
- Repository 패턴: `domains/adapter/output/infrastructure/mongo/repository/repository.md` 참조
- 네이밍 컨벤션: `rules/mongodb.md` 참조
