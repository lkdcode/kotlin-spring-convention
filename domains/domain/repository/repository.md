# domain/repository — Repository Interface

도메인이 인프라를 모르게 하는 포트. 인터페이스만 선언.

## ⚠️ 도입 기준

**단순 CRUD 에서는 생략한다.**
`application/output/` 의 `CommandPort` / `QueryPort` 로 충분.

도입이 필요한 경우:
- Domain Service 가 여러 Aggregate 를 조합해 규칙을 실행할 때
- 도메인 규칙 자체가 영속 데이터 조회에 의존할 때

```kotlin
// ✅ 도입 의미 있음 — Domain Service 가 Aggregate 직접 조회
class TransferService(
    private val accountRepository: AccountRepository,
) {
    fun transfer(sourceId: AccountId, targetId: AccountId, amount: Money) {
        val source = accountRepository.getById(sourceId)
        val target = accountRepository.getById(targetId)
        source.withdraw(amount)
        target.deposit(amount)
    }
}

// ❌ 도입 불필요 — Application 계층 CommandPort 로 충분
class CreateOrderService(
    private val commandPort: OrderCommandPort,  // 이걸로 충분
) : CreateOrderUsecase { ... }
```

```kotlin
// Aggregate Root 단위로 선언
interface OrderRepository {
    fun save(order: Order): Order
    fun getById(id: OrderId): Order              // 없으면 throw DomainException
    fun findById(id: OrderId): Order?            // 없으면 null
    fun findAllByCustomerId(customerId: CustomerId): List<Order>
    fun delete(id: OrderId)
    fun existsById(id: OrderId): Boolean
}

interface ProductRepository {
    fun save(product: Product): Product
    fun getById(id: ProductId): Product
    fun findAll(): List<Product>
}
```

## QueryPort 와 구분

| | Repository (domain) | QueryPort (application/output) |
|---|---|---|
| 위치 | `domain/repository/` | `application/output/query/` |
| 반환 | Aggregate / Entity | Query DTO (화면용) |
| 목적 | 도메인 조작 (저장/수정/삭제) | 화면 조회 (페이징, 필터, JOIN) |
| 의존 | 도메인 모델 | DTO, Pagination |
| 구현 | JPA Repository | QueryDSL Adapter |

```
Command 흐름: CommandApi → Usecase → Repository → Entity 조작
Query  흐름:  QueryApi  → Usecase → QueryPort  → DTO 반환
```

## 규칙
- **Aggregate Root 단위로만** 선언 — 내부 Entity 는 Root 통해 접근
- `getById` — 없으면 `DomainException` throw
- `findById` — 없으면 `null` 반환
- 컬렉션 조회는 빈 List 반환 (`null` 금지)
- 구현체는 `adapter/output/command/*CommandAdapter` 또는 별도 `*RepositoryAdapter`
