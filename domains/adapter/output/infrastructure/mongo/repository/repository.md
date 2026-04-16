# adapter/output/infrastructure/mongo/repository — MongoDB Repository

## Spring MVC

```kotlin
interface FooDocumentRepository : MongoRepository<FooDocument, String> {
    fun findTop10ByOrderByCreatedAtDesc(): List<FooDocument>

    @Query(value = "{}", fields = "{ 'foo_name': 1 }")
    fun findAllFooNames(): List<FooDocument>
}

// loadById 확장함수 패턴
fun FooDocumentRepository.loadById(id: String): FooDocument =
    findById(id).orElseThrow { throw ApiException(ApiResponseCode.FOO_NOT_FOUND) }
```

## Spring WebFlux

```kotlin
interface FooDocumentRepository : ReactiveMongoRepository<FooDocument, String>

fun FooDocumentRepository.loadById(id: String): Mono<FooDocument> =
    findById(id).switchIfEmpty(Mono.error(ApiException(ApiResponseCode.FOO_NOT_FOUND)))
```

---

## 규칙

- 인터페이스명: `*DocumentRepository`
- `loadById` 확장함수 패턴 — JPA Repository 와 동일
- MVC: `MongoRepository`, WebFlux: `ReactiveMongoRepository`
- Document 클래스 규칙 → `mongo/doc/doc.md` 참조
