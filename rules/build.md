# rules/build — Gradle 의존성 관리 (buildSrc)

의존성을 `buildSrc` 에서 중앙 관리. 모든 모듈은 `apply { from() }` 으로 주입받아 사용.

---

## 디렉토리 구조

```
buildSrc/
└── src/main/kotlin/
    ├── BuildSet.kt          ← 버전 상수 (SpringBoot, Kotlin, Versioning)
    ├── SpringBootLibs.kt    ← spring-boot-dep.gradle 경로 + 의존성 상수
    ├── KotlinLibs.kt        ← kotlin-dep.gradle 경로 + 의존성 상수
    ├── DatabaseLibs.kt      ← database-dep.gradle 경로 + Flyway/DB 상수
    ├── SpringSecurityLibs.kt← security-dep.gradle 경로 + JWT 상수
    ├── CacheLibs.kt         ← cache-dep.gradle 경로 + Caffeine 상수
    ├── QueryDslLibs.kt      ← query-dsl-dep.gradle 경로 + QueryDSL 상수
    ├── ApiDocsLibs.kt       ← api-docs-dep.gradle 경로 + Swagger 상수
    ├── AwsLibs.kt           ← aws-sdk.gradle 경로 + AWS SDK 상수
    └── TestLibs.kt          ← test-dep.gradle 경로 + Test 상수

gradle/config/
    ├── spring-boot-dep.gradle
    ├── kotlin-dep.gradle
    ├── database-dep.gradle
    ├── security-dep.gradle
    ├── cache-dep.gradle
    ├── query-dsl-dep.gradle
    ├── api-docs-dep.gradle
    ├── aws-sdk.gradle
    └── test-dep.gradle
```

---

## BuildSet.kt — 버전 상수

```kotlin
object SpringBoot {
    const val ID = "org.springframework.boot"
    const val VERSION = "3.4.5"
    const val DEPENDENCY_MANAGEMENT = "io.spring.dependency-management"
    const val DEPENDENCY_MANAGEMENT_VERSION = "1.1.7"
}

object Kotlin {
    const val KOTLIN_VERSION = "2.1.10"
    const val WITH_JVM = "jvm"
    const val KAPT = "kapt"
    const val WITH_SPRING = "plugin.spring"
    const val WITH_JPA = "plugin.jpa"
    const val WITH_LOMBOK = "plugin.lombok"
}

object Versioning {
    const val JAVA = 21
    const val GROUP = "com.back"
    const val VERSION = "v1.0.0"
}
```

---

## *Libs.kt 패턴

각 파일은 **`PATH` 상수** (gradle/config 경로) + **의존성 문자열 상수** 로 구성.

```kotlin
// 예: SpringBootLibs.kt
object SpringBootLibs {
    const val PATH = "gradle/config/spring-boot-dep.gradle"  // ← 경로

    const val WEB = "org.springframework.boot:spring-boot-starter-web"
    const val DATA_JPA = "org.springframework.boot:spring-boot-starter-data-jpa"
    const val VALIDATION = "org.springframework.boot:spring-boot-starter-validation"
    const val SECURITY = "org.springframework.boot:spring-boot-starter-security"
    const val QUARTZ = "org.springframework.boot:spring-boot-starter-quartz"
    const val TEST = "org.springframework.boot:spring-boot-starter-test"
}
```

---

## gradle/config/*.gradle 패턴

buildSrc 상수를 사용해 의존성 선언.

```groovy
// spring-boot-dep.gradle
dependencies {
    implementation(SpringBootLibs.WEB)
    implementation(SpringBootLibs.DATA_JPA)
    implementation(SpringBootLibs.VALIDATION)
    implementation(SpringBootLibs.SECURITY)
    testImplementation(SpringBootLibs.TEST)
}
```

```groovy
// kotlin-dep.gradle
dependencies {
    implementation(KotlinLibs.JACKSON_MODULE_KOTLIN)
    implementation(KotlinLibs.KOTLIN_REFLECT)
}

kotlin {
    compilerOptions {
        freeCompilerArgs.addAll(KotlinLibs.JSR305_STRICT_MODE)
    }
}

noArg {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

```groovy
// security-dep.gradle
dependencies {
    implementation(SpringSecurityLibs.JJWT_API)
    runtimeOnly(SpringSecurityLibs.JJWT_IMPL)
    runtimeOnly(SpringSecurityLibs.JJWT_JACKSON)
}
```

---

## build.gradle.kts — 모듈에서 주입

```kotlin
plugins {
    id(SpringBoot.ID) version SpringBoot.VERSION
    id(SpringBoot.DEPENDENCY_MANAGEMENT) version SpringBoot.DEPENDENCY_MANAGEMENT_VERSION

    kotlin(Kotlin.WITH_JVM) version Kotlin.KOTLIN_VERSION
    kotlin(Kotlin.KAPT) version Kotlin.KOTLIN_VERSION
    kotlin(Kotlin.WITH_SPRING) version Kotlin.KOTLIN_VERSION
    kotlin(Kotlin.WITH_JPA) version Kotlin.KOTLIN_VERSION
}

group = Versioning.GROUP
version = Versioning.VERSION

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(Versioning.JAVA)
    }
}

apply {
    from(SpringBootLibs.PATH)    // ← gradle/config/spring-boot-dep.gradle 주입
    from(KotlinLibs.PATH)
    from(DatabaseLibs.PATH)
    from(SpringSecurityLibs.PATH)
    from(CacheLibs.PATH)
    from(ApiDocsLibs.PATH)
    from(TestLibs.PATH)
    // 필요한 것만 선택 적용
}
```

---

## 주요 의존성 상수 (예시)

아래는 자주 사용되는 Libs 클래스 예시. 새 의존성이 필요하면 동일한 패턴으로 `*Libs.kt` 를 추가하면 됨.

| Libs 클래스 | 주요 내용 |
|---|---|
| `SpringBootLibs` | web, jpa, security, validation, quartz |
| `KotlinLibs` | jackson, reflect, coroutines, JSR305 |
| `DatabaseLibs` | flyway, DB 드라이버 |
| `SpringSecurityLibs` | jjwt-api/impl/jackson |
| `CacheLibs` | caffeine |
| `QueryDslLibs` | querydsl-jpa, kapt 설정 |
| `ApiDocsLibs` | springdoc-openapi (Swagger) |
| `AwsLibs` | aws-sdk s3 |
| `TestLibs` | spring-boot-starter-test |
| `RedisLibs` | spring-boot-starter-data-redis (필요 시 추가) |
| `KafkaLibs` | spring-kafka (필요 시 추가) |
| `JooqLibs` | jooq, jooq-kotlin, codegen (jOOQ 섹션 참조) |
| `FlywayLibs` | flyway-core, DB 방언 (jOOQ 섹션 참조) |

---

---

## jOOQ + Flyway — DB 접속정보 주입

jOOQ 코드 생성은 빌드 타임에 실행 → DB 에 직접 접속 필요.
**`YmlService`** (buildSrc Groovy) 가 `application-{profile}.yml` 을 읽어 주입.

### 실행 순서

```
flywayMigrate        ← DB 스키마 마이그레이션
    ↓
generateJooq         ← 마이그레이션된 스키마 기반으로 코드 생성
    ↓
build                ← 전체 빌드
```

```kotlin
// build.gradle.kts
tasks.named("generateJooq") { dependsOn("flywayMigrate") }
tasks.named("build") { dependsOn("generateJooq") }
```

### buildSrc — Libs 상수

```kotlin
// FlywayLibs.kt
object FlywayLibs {
    const val ID = "org.flywaydb.flyway"
    const val VERSION = "11.15.0"
    const val PATH = "gradle/dep/flyway-dep.gradle"
    const val CONFIG_PATH = "gradle/config/flyway-config.gradle"
    const val CORE = "org.flywaydb:flyway-core:$VERSION"
    const val DB_POSTGRESQL = "org.flywaydb:flyway-database-postgresql:$VERSION"
    const val POSTGRESQL_CONNECTOR = "org.postgresql:postgresql:42.7.4"
}

// JooqLibs.kt
object JooqLibs {
    const val PATH = "gradle/dep/jooq-dep.gradle"
    const val CONFIG_PATH = "gradle/config/jooq-config.gradle"
    const val JOOQ_VERSION = "3.20.8"
    const val TARGET_PACKAGE_NAME = "{project}.jooq"
    const val DEFAULT_DIRECTORY = "generated/jooq"
    const val STRATEGY_NAME = "{project}.framework.config.JPrefixGeneratorStrategy"
    const val JOOQ = "org.jooq:jooq:$JOOQ_VERSION"
    const val JOOQ_KOTLIN = "org.jooq:jooq-kotlin:$JOOQ_VERSION"
    const val JOOQ_META = "org.jooq:jooq-meta:$JOOQ_VERSION"
    const val JOOQ_CODEGEN = "org.jooq:jooq-codegen:$JOOQ_VERSION"
}
```

### buildSrc — YmlService (DB 접속정보 추출)

`application-{profile}.yml` 의 `spring.flyway.*` 를 빌드 타임에 읽어 주입.
프로파일은 `-Pspring.profiles.active=local` 로 전달 (기본값: `local`).

```groovy
// buildSrc/src/main/groovy/.../YmlService.groovy
class YmlService {
    YmlService(Project project) { this.project = project }

    String getFlywayUrl()      { navigate(loadYml(), "spring", "flyway", "url") }
    String getFlywayUser()     { navigate(loadYml(), "spring", "flyway", "user") }
    String getFlywayPassword() { navigate(loadYml(), "spring", "flyway", "password") }
    String getFlywaySchema()   { navigate(loadYml(), "spring", "flyway", "schemas") }
    List   getFlywayLocations(){ navigate(loadYml(), "spring", "flyway", "locations") }

    private Map loadYml() {
        def profile = project.findProperty("spring.profiles.active") ?: "local"
        def file = project.file("src/main/resources/application-${profile}.yml")
        return new Yaml().load(file.text) as Map ?: [:]
    }
}
```

### application-{profile}.yml — 단일 소스

```yaml
# application-local.yml
spring:
  flyway:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://localhost:5432/mydb
    user: myuser
    password: mypassword
    schemas: public
    locations: classpath:db/migration
```

DB 접속정보는 `application-{profile}.yml` 에서만 관리. `build.gradle.kts` 에 하드코딩 금지.

### gradle/config/flyway-config.gradle

```groovy
import {project}.yml.YmlService

def yml = new YmlService(project)

flyway {
    driver   = yml.getFlywayDriver()
    url      = yml.getFlywayUrl()
    user     = yml.getFlywayUser()
    password = yml.getFlywayPassword()
    defaultSchema = yml.getFlywaySchema()
    locations     = yml.getFlywayLocations()
    cleanDisabled = false
}

tasks.named('flywayMigrate') { dependsOn processResources }
```

### gradle/config/jooq-config.gradle

```groovy
import {project}.yml.YmlService

ext['jooq.version'] = JooqLibs.JOOQ_VERSION
def yml = new YmlService(project)

jooq {
    version = JooqLibs.JOOQ_VERSION
    configurations {
        main {
            generateSchemaSourceOnCompilation = true
            jooqConfiguration.with {
                jdbc.with {
                    driver   = 'org.postgresql.Driver'
                    url      = yml.getFlywayUrl()
                    user     = yml.getFlywayUser()
                    password = yml.getFlywayPassword()
                }
                generator.with {
                    name = 'org.jooq.codegen.KotlinGenerator'
                    strategy.name = JooqLibs.STRATEGY_NAME
                    database.with {
                        name = 'org.jooq.meta.postgres.PostgresDatabase'
                        inputSchema = 'public'
                        excludes = 'flyway_schema_history'
                    }
                    generate.with {
                        javaTimeTypes = true
                        records = true
                        daos = false
                        kotlinNotNullPojoAttributes = true
                    }
                    target.with {
                        packageName = JooqLibs.TARGET_PACKAGE_NAME
                        directory = layout.buildDirectory.dir(JooqLibs.DEFAULT_DIRECTORY).get().asFile.path
                    }
                }
            }
        }
    }
}
```

### build.gradle.kts — 플러그인 + 적용

```kotlin
plugins {
    id(Flyway.ID) version Flyway.VERSION
    id(JOOQ.NS_STUDER_JOOQ) version JOOQ.VERSION
    // ...
}

apply {
    from(FlywayLibs.PATH)
    from(FlywayLibs.CONFIG_PATH)
    from(JooqLibs.PATH)
    from(JooqLibs.CONFIG_PATH)
}

tasks.named("generateJooq") { dependsOn("flywayMigrate") }
tasks.named("build") { dependsOn("generateJooq") }
```

### JPrefixGeneratorStrategy — 멀티 모듈 전용

멀티 모듈에서 jOOQ 생성 클래스명에 `J` prefix 를 붙여 JPA Entity 와 이름 충돌 방지.
별도 `jooq-config` 모듈로 분리 (`settings.gradle.kts` 에 `include("jooq-config")`).

```kotlin
// jooq-config/src/main/kotlin/.../JPrefixGeneratorStrategy.kt
class JPrefixGeneratorStrategy : DefaultGeneratorStrategy() {
    override fun getJavaClassName(definition: Definition, mode: GeneratorStrategy.Mode): String =
        if (mode == Mode.DEFAULT || mode == Mode.RECORD) {
            "J" + super.getJavaClassName(definition, mode)
        } else super.getJavaClassName(definition, mode)

    override fun getJavaIdentifier(definition: Definition?): String =
        when (definition) {
            is TableDefinition -> "J" + super.getJavaIdentifier(definition)
            else -> super.getJavaIdentifier(definition)
        }
}
```

```kotlin
// jooq-config/build.gradle.kts
plugins { kotlin(Kotlin.WITH_JVM) version Kotlin.KOTLIN_VERSION }
dependencies { implementation(JooqLibs.JOOQ_CODEGEN) }
```

`jooq-config.gradle` 에서 `strategy.name = JooqLibs.STRATEGY_NAME` 으로 참조.

### 코드 생성 실행

```bash
# local 프로파일 (기본값)
./gradlew generateJooq

# dev 프로파일
./gradlew generateJooq -Pspring.profiles.active=dev
```

---

---

## 커스텀 태스크 관리

| 종류 | 위치 | 예시 |
|---|---|---|
| 태스크 와이어링 (`dependsOn`) | `build.gradle.kts` | `generateJooq dependsOn flywayMigrate` |
| 단순 커스텀 태스크 (5줄 이하) | `build.gradle.kts` | `flywayCleanLocal` |
| 복잡/재사용 태스크 | `gradle/tasks/*.gradle` | 환경별 배포 스크립트 |

```kotlin
// build.gradle.kts — 태스크 와이어링 + 단순 커스텀 태스크

tasks.named("generateJooq") { dependsOn("flywayMigrate") }
tasks.named("build") { dependsOn("generateJooq") }
tasks.named("bootJar") { dependsOn("test") }

tasks.withType<Test> { useJUnitPlatform() }

tasks.named("clean") {
    doLast { generated.deleteRecursively() }
}

// 단순 커스텀 태스크 — build.gradle.kts 에 직접
tasks.register("flywayCleanLocal") {
    group = "flyway"
    description = "Flyway clean + migrate (local)"
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

## 단일 모듈 vs 멀티 모듈

### 단일 모듈

```
project/
├── buildSrc/
├── gradle/
├── src/
└── build.gradle.kts    ← 플러그인 선언 + 적용 모두 여기
```

```kotlin
// build.gradle.kts (단일 모듈)
plugins {
    id(SpringBoot.ID) version SpringBoot.VERSION          // 직접 적용
    kotlin(Kotlin.WITH_JVM) version Kotlin.KOTLIN_VERSION
}

group = Versioning.GROUP
version = Versioning.VERSION
```

---

### 멀티 모듈

```
project/
├── buildSrc/
├── api-module/
│   ├── gradle/
│   └── build.gradle.kts    ← 서브모듈 — 플러그인 선택적 적용
├── jooq-config/
│   └── build.gradle.kts
├── batch-module/
│   └── build.gradle.kts
├── build.gradle.kts         ← 루트 — 플러그인 선언만 (apply false)
└── settings.gradle.kts      ← 모듈 등록
```

**settings.gradle.kts** — 모듈 등록

```kotlin
rootProject.name = "my-project"

include("api-module")
include("jooq-config")
include("batch-module")

pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}
```

**루트 build.gradle.kts** — 플러그인 선언 (`apply false`), 공통 설정

```kotlin
buildscript {
    repositories { mavenCentral() }
    dependencies {
        classpath("org.postgresql:postgresql:42.7.4")  // Flyway 드라이버
    }
}

plugins {
    id(SpringBoot.ID) version SpringBoot.VERSION apply false          // 선언만
    id(SpringBoot.DEPENDENCY_MANAGEMENT) version SpringBoot.DEPENDENCY_MANAGEMENT_VERSION apply false
    id(Flyway.ID) version Flyway.VERSION apply false
    id(JOOQ.NS_STUDER_JOOQ) version JOOQ.VERSION apply false
    kotlin(Kotlin.WITH_JVM) version Kotlin.KOTLIN_VERSION apply false
    kotlin(Kotlin.WITH_SPRING) version Kotlin.KOTLIN_VERSION apply false
}

allprojects {
    group = Versioning.GROUP
    version = Versioning.VERSION
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
}
```

**서브모듈 build.gradle.kts** — 필요한 플러그인만 선택 적용

```kotlin
plugins {
    id(SpringBoot.ID)                     // 버전 생략 — 루트에서 선언됨
    id(SpringBoot.DEPENDENCY_MANAGEMENT)
    id(Flyway.ID)
    id(JOOQ.NS_STUDER_JOOQ)
    kotlin(Kotlin.WITH_JVM)
    kotlin(Kotlin.WITH_SPRING)
}

apply {
    from(SpringBootLibs.PATH)
    from(JooqLibs.PATH)
    from(JooqLibs.CONFIG_PATH)
    from(FlywayLibs.PATH)
    from(FlywayLibs.CONFIG_PATH)
}
```

### 단일 vs 멀티 선택 기준

| | 단일 모듈 | 멀티 모듈 |
|---|---|---|
| 프로젝트 규모 | 소~중형 | 중~대형 |
| 빌드 분리 | 불필요 | jooq-config, batch 등 독립 빌드 필요 |
| 플러그인 관리 | 단순 | 루트 선언 + 서브모듈 선택 적용 |
| 권장 시작 방식 | 단일 모듈로 시작 | 필요 시 분리 |

---

## 규칙

- 의존성 버전은 `buildSrc` 에서만 관리 — `build.gradle.kts` 에 버전 직접 작성 금지
- 새 의존성 추가: `*Libs.kt` 상수 추가 → `gradle/config/*.gradle` 선언 → `apply { from() }` 적용
- 새 Libs 파일 추가 시 `PATH` 상수 필수 — `gradle/config/` 경로와 매핑
- DB 접속정보는 `application-{profile}.yml` 에서만 관리 — `build.gradle.kts` 하드코딩 금지
- jOOQ 코드 생성은 반드시 Flyway 마이그레이션 이후 (`generateJooq` dependsOn `flywayMigrate`)
- 멀티 모듈 루트: 플러그인은 `apply false` 로 선언만 — 서브모듈에서 선택 적용
- 단순 커스텀 태스크는 `build.gradle.kts` 에 직접 — 복잡한 태스크는 `gradle/tasks/` 분리
