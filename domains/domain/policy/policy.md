# domain/policy — Policy

비즈니스 정책에 따른 계산/변환 로직. 정책이 바뀌면 구현체만 교체.

```kotlin
// 정책 인터페이스
interface DiscountPolicy {
    fun calculate(order: Order): Money
}

// 구현체
object NoDiscountPolicy : DiscountPolicy {
    override fun calculate(order: Order) = Money.ZERO
}

class RateDiscountPolicy(private val rate: BigDecimal) : DiscountPolicy {
    override fun calculate(order: Order) =
        Money(order.totalPrice().amount * rate)
}

class FixedDiscountPolicy(private val amount: Money) : DiscountPolicy {
    override fun calculate(order: Order) = amount
}
```

## 상태 없는 단순 정책 — object

```kotlin
object WeeklyAggregationPolicy {
    fun groupByWeek(items: List<DailyRecord>): Map<Int, List<DailyRecord>> =
        items.groupBy { it.date.get(WeekFields.ISO.weekOfWeekBasedYear()) }

    fun weekRange(year: Int, week: Int): ClosedRange<LocalDate> {
        val start = LocalDate.ofYearDay(year, 1)
            .with(WeekFields.ISO.weekOfWeekBasedYear(), week.toLong())
            .with(WeekFields.ISO.dayOfWeek(), 1)
        return start..start.plusDays(6)
    }
}
```

## Spec vs Policy 구분

| | Spec | Policy |
|---|---|---|
| 반환 | `Boolean` | 계산 결과값 |
| 역할 | 조건 판단 (가능/불가능) | 값 계산/변환 |
| 예시 | `isSatisfiedBy()` | `calculate()`, `groupBy()` |
| 교체 | 조합으로 확장 | 구현체 교체로 변경 |

## 규칙
- 정책 변경 가능성이 있으면 인터페이스로 추상화
- 단일 정책이면 `object` 로 선언
- 외부 의존성 없음 — 외부 데이터(DB, API)가 필요한 경우 Application Service 에서 조회 후 Policy 에 파라미터로 전달
