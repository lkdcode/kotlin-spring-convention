# application/services/command — *CommandService.kt / *Service.kt (구현체)

```kotlin
// command 1개 — execute()
@Service
@Transactional
class CreateFooService(
    private val commandPort: FooCommandPort,
    private val validator: FooValidator,
) : CreateFooUsecase {

    override fun execute(model: CreateFooModel) {
        validator.validateDuplicate(model.code)
        commandPort.create(model)
    }
}

// command 2개 이상 — 메서드명 명시
@Service
@Transactional
class FooCommandService(
    private val commandPort: FooCommandPort,
    private val validator: FooValidator,
    private val guard: FooGuard,
) : FooCommandUsecase {

    override fun create(model: CreateFooModel) {
        validator.validateDuplicate(model.code)
        commandPort.create(model)
    }

    override fun delete(ids: List<Long>) {
        guard.requireCanDelete(ids)
        commandPort.delete(ids)
    }
}
```

## 규칙

- **호출 순서와 트랜잭션 제어**의 책임
- `@Transactional` 클래스 레벨 사용 — 단, 무거운 CPU 연산(인코딩, 대용량 변환 등)이 DB 작업 앞에 있으면 해당 연산을 트랜잭션 밖으로 분리
- `application/ports/output/` 의 포트만 주입 — **adapter 계층 코드 import 절대 금지**
- 타 도메인의 `ports/output/*` 참조 가능 — 도메인 간 Guard, Validator, Checker 등 포트 인터페이스를 통해 활용
- `domain` 사용 적극 활용 — `spec`, `policy`, `service` 등
- 필요 시 fire-and-forget 패턴 사용 가능
- **기본은 포트 직접 호출** — DSL 패턴 사용 금지
- DSL 도입이 합당하다고 판단되면 **반드시 사용자에게 먼저 제안하고 승인 후 진행**

## DSL 도입 판단 기준 (제안 전 확인)

아래 조건을 **모두** 만족할 때만 사용자에게 제안 가능:

1. 단계가 3개 이상이고
2. 단계 간 데이터가 흘러야 하며 (이전 단계 결과를 다음 단계가 사용)
3. 단계 조합이 변경될 가능성이 있을 때

```
// 제안 예시
"이 작업은 validate → enrich → save 단계가 있고 각 단계 결과가 연결됩니다.
DSL 패턴을 적용하면 흐름이 명시적으로 표현됩니다. 적용할까요?"
```

조건 미충족 시 직접 호출로 작성.

## 위치 참조

| 종류 | 위치 |
|---|---|
| Command Model | `ports/input/command/model/` |
| 일급 컬렉션 (도메인) | `domain/model/vo.md` |
| Query 공통 규약 | `ports/input/query/common/` |
