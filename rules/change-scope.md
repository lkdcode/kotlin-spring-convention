# rules/change-scope — fix / refactor / feat 규칙

변경의 범위와 목적을 명확히 구분. 하나의 작업에 하나의 목적만.

---

## fix — 버그 수정

**오직 버그만 수정한다.**

```
✅ 허용
- 버그를 일으키는 코드만 수정
- 버그와 직접 연관된 테스트 추가

❌ 금지
- 기능 추가
- 관련 없는 코드 리팩토링
- 스타일 변경
```

> fix 작업 중 리팩토링이 필요하다면 별도 커밋으로 분리.

---

## refactor — 리팩토링

**동작은 변경하지 않는다. 구조만 개선한다.**

```
✅ 허용
- 코드 구조 개선 (메서드 분리, 클래스 분리)
- 네이밍 개선
- 중복 제거
- 성능 개선 (외부 동작 동일)

❌ 금지
- 기능 추가 또는 제거
- 버그 수정 (별도 fix 로 분리)
- API 스펙 변경
```

> 리팩토링 전후 동작이 동일해야 한다.

---

## feat — 기능 추가

**새로운 기능을 추가한다.**

```
✅ 허용
- 새로운 엔드포인트, UseCase, 도메인 로직 추가
- 기능에 필요한 리팩토링 동반 (범위 최소화)

❌ 금지
- 기존 버그를 함께 수정 (별도 fix 로 분리)
- 관련 없는 코드 변경
```

---

## 공통 규칙

| | fix | refactor | feat |
|---|---|---|---|
| 동작 변경 | 최소화 | ❌ | ✅ |
| 기능 추가 | ❌ | ❌ | ✅ |
| 버그 수정 | ✅ | ❌ | ❌ |
| 구조 변경 | 최소화 | ✅ | 허용 (최소) |

- 목적이 섞이면 커밋을 분리
- 하나의 PR = 하나의 목적

---

## 사이드 이펙트 체크

**OutputPort 와 UseCase 는 여러 도메인에서 공유될 수 있다.**
도메인 정책/규칙이 UseCase 와 OutputPort 를 통해 동작하기 때문에,
하나를 변경하면 이를 사용하는 모든 곳에 영향을 준다.

```
변경 전 반드시 확인:
- 이 OutputPort / UseCase 를 사용하는 곳이 어디인가?
- 변경이 다른 도메인의 동작에 영향을 주지 않는가?
- 도메인 정책(spec, policy)과 일치하는가?
```

---

## 중복 생성 금지

### OutputPort 중복 금지
동일한 기능의 OutputPort 가 이미 있으면 새로 만들지 않는다.
도메인마다 흩어져 있으면 안 되고, 재사용 가능한 포트는 공유한다.

```kotlin
// ❌ 동일 기능을 도메인마다 중복 선언
interface FooExistenceChecker { fun exists(id: Long): Boolean }
interface BarExistenceChecker { fun exists(id: Long): Boolean }

// ✅ 공통 포트 하나로 통합 또는 타 도메인 포트 재사용
```

### UseCase 중복 금지
단일 액터의 단일 행위에 대한 UseCase 는 하나만 존재해야 한다.
동일한 UseCase 가 여러 개면 어느 것을 사용해야 하는지 혼란이 생긴다.

```kotlin
// ❌ 동일 행위의 UseCase 중복
interface CreateFooUsecase { fun execute(model: CreateFooModel) }
interface RegisterFooUsecase { fun execute(model: CreateFooModel) }  // 동일 행위

// ✅ 하나만 유지
interface CreateFooUsecase { fun execute(model: CreateFooModel) }
```

---

## 응집도 / 결합도

- **응집도 높게**: 같은 도메인의 관련 코드는 한 곳에
- **결합도 낮게**: 도메인 간 직접 의존 금지 — 포트 인터페이스를 통해 간접 의존
- 도메인 비즈니스 정책(`domain/spec`, `domain/policy`)이 UseCase 와 OutputPort 의 동작 기준
