# adapter/input/schedule — Scheduler

| 상황 | 선택 |
|---|---|
| 기본 | Quartz |
| 무겁고 오래 걸리는 대용량 작업 | Spring Batch |

---

## Quartz Scheduler

## 디렉토리 구조

```
adapter/input/schedule/
├── {JobName}Schedule.kt       ← Job 구현체
└── config/
    └── {JobName}Config.kt     ← JobDetail + Trigger + Cron interface + @Profile 구현체
```

> `spring-boot-starter-quartz` 가 `SpringBeanJobFactory` 를 자동 구성 — 생성자 주입 그대로 동작.

---

## Job 구현체 — *Schedule.kt

```kotlin
@DisallowConcurrentExecution
class FooSchedule(
    private val scheduleInput: FooScheduleInput,
) : Job {

    override fun execute(context: JobExecutionContext) {
        withLogging(TARGET) {
            scheduleInput.execute()
        }
    }

    companion object {
        private const val TARGET =
            "🔵 FooSchedule [테이블] foos"
    }
}
```

## Config — *Config.kt

JobDetail + Trigger + Cron interface + @Profile 구현체를 한 파일에.

```kotlin
@Configuration
class FooScheduleConfig(
    private val cron: CronFooSchedule,
) {
    @Bean
    fun getFooScheduleJobDetail(): JobDetail =
        JobBuilder.newJob(FooSchedule::class.java)
            .withIdentity(IDENTITY)
            .storeDurably()
            .build()

    @Bean
    fun getFooScheduleTrigger(): Trigger =
        TriggerBuilder.newTrigger()
            .forJob(getFooScheduleJobDetail())
            .withIdentity(IDENTITY)
            .withSchedule(cron.getCronSchedule())
            .startNow()
            .build()

    companion object {
        private val IDENTITY = FooSchedule::class.java.simpleName
    }
}

// Cron interface + @Profile 구현체 — 같은 파일 하단에 선언
interface CronFooSchedule {
    fun getCronSchedule(): CronScheduleBuilder
}

@Profile(value = ["dev"])
@Component
class CronFooScheduleDev : CronFooSchedule {
    override fun getCronSchedule(): CronScheduleBuilder =
        CronScheduleBuilder.cronSchedule("0 1 0/1 ? * * *")
}

@Profile(value = ["prod"])
@Component
class CronFooScheduleProd : CronFooSchedule {
    override fun getCronSchedule(): CronScheduleBuilder =
        CronScheduleBuilder.cronSchedule("0 1 0/1 ? * * *")
}

@Profile(value = ["local"])
@Component
class CronFooScheduleLocal : CronFooSchedule {
    override fun getCronSchedule(): CronScheduleBuilder =
        CronScheduleBuilder.cronSchedule("0 10/10 * * * ? 2030")  // local 실행 방지
}
```

---

## withLogging — 스케줄러 전용 로깅

`framework/schedule/withLogging.kt` 에 위치. 모든 Job 에서 사용.

```kotlin
inline fun <reified T> T.withLogging(target: String, block: () -> Unit) {
    val startTime = TimeUtil.nowKST()
    try {
        this.logInfo("START $target")
        block()
        this.logInfo("END-- $target Latency: ${between(startTime, TimeUtil.nowKST()).toMillis()}")
    } catch (e: Exception) {
        this.logWarn(e.toString())
        this.logInfo("🚫 FAIL $target")
    }
}
```

### TARGET 작성 형식

```kotlin
companion object {
    private const val TARGET =
        "🔵 {ClassName} [테이블] {table_name}"
}
```

| 항목 | 설명 | 예시 |
|---|---|---|
| 이모지 | 시각적 구분 | `🔵` |
| ClassName | Job 클래스명 | `FooSchedule` |
| [테이블] | 대상 테이블 태그 | `[테이블]` |
| table_name | 실제 DB 테이블명 | `foos` |

### 로그 출력 예시

```
INFO  START 🔵 FooSchedule [테이블] foos
INFO  END-- 🔵 FooSchedule [테이블] foos Latency: 1234
WARN  java.lang.RuntimeException: ...
INFO  🚫 FAIL 🔵 FooSchedule [테이블] foos
```

---

## 규칙

- `@DisallowConcurrentExecution` 필수 — 중복 실행 방지
- `execute()` 파라미터 `?` 제거
- UseCase 만 호출 — 비즈니스 로직 직접 작성 금지
- Cron 표현식은 `cron/` 에 interface + `@Profile` 구현체로 분리
- Profile 은 `dev`, `prod`, `local` 3개로 분리
- `local` 프로필은 먼 미래 날짜로 실수 실행 방지
- `withLogging()` 필수 — 모든 Job 에 적용
- TARGET 형식: `"🔵 {ClassName} [테이블] {table_name}"`
