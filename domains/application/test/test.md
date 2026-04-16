# application/test — Application Service 테스트

Spring Context 없음. MockK 로 Output Port 를 mocking 하여 UseCase 흐름 검증.

---

## 대상

| 대상 | 테스트 내용 |
|---|---|
| *CommandService | Port 호출 순서, 예외 분기 |
| *QueryService | QueryPort 호출, 반환값 매핑 |

---

## Command Service 테스트

```kotlin
class CreateFooCommandServiceTest : DescribeSpec({

    val mockCommandPort = mockk<FooCommandPort>()
    val mockValidator = mockk<FooValidator>()

    val service = CreateFooCommandService(
        commandPort = mockCommandPort,
        validator = mockValidator,
    )

    beforeTest { clearMocks(mockCommandPort, mockValidator) }

    describe("`execute` 메서드는") {
        context("유효한 CreateFooModel 이 전달되면") {
            it("validator 를 호출하고 저장할 것이다") {
                // Given
                val model = CreateFooModelFactory.create()
                every { mockValidator.validateDuplicate(any()) } just Runs
                every { mockCommandPort.create(any()) } returns 1L

                // When
                service.execute(model)

                // Then
                verify(exactly = 1) { mockValidator.validateDuplicate(model.code) }
                verify(exactly = 1) { mockCommandPort.create(model) }
            }
        }

        context("중복된 코드가 존재하면") {
            it("ApiException 을 던질 것이다") {
                // Given
                val model = CreateFooModelFactory.create()
                every { mockValidator.validateDuplicate(any()) } throws
                    ApiException(ApiResponseCode.FOO_DUPLICATE_CODE)

                // When & Then
                val exception = shouldThrow<ApiException> {
                    service.execute(model)
                }
                exception.getApiResponseCode() shouldBe ApiResponseCode.FOO_DUPLICATE_CODE

                verify(exactly = 1) { mockValidator.validateDuplicate(model.code) }
                verify(exactly = 0) { mockCommandPort.create(any()) }
            }
        }
    }
})
```

## Query Service 테스트

```kotlin
class FetchFooQueryServiceTest : DescribeSpec({

    val mockQueryPort = mockk<FooQueryPort>()

    val service = FetchFooQueryService(
        queryPort = mockQueryPort,
    )

    beforeTest { clearMocks(mockQueryPort) }

    describe("`getById` 메서드는") {
        context("존재하는 ID 가 전달되면") {
            it("FetchFooDTO 를 반환할 것이다") {
                // Given
                val expected = FetchFooDTOFactory.create()
                every { mockQueryPort.getById(any()) } returns expected

                // When
                val result = service.getById(FooId(1L))

                // Then
                result shouldBe expected
            }
        }
    }

    describe("`fetchList` 메서드는") {
        context("조건이 전달되면") {
            it("페이지네이션 결과를 반환할 것이다") {
                // Given
                val expected = FetchFooDTOListFactory.create()
                every { mockQueryPort.fetchList(any(), any()) } returns expected

                // When
                val result = service.fetchList(emptyList(), Pageable.ofSize(15))

                // Then
                result shouldBe expected
            }
        }
    }
})
```

---

## 규칙

- Spring Context 사용 금지 — 순수 단위 테스트
- 모든 Output Port 는 `mockk<>()` 로 mocking
- `beforeTest { clearMocks() }` 필수
- 실제 구현체(Adapter) 주입 금지
- `verify()` 로 Port 호출 횟수 검증 — 특히 예외 발생 시 이후 Port 미호출 확인
- `// Given`, `// When`, `// Then` 주석으로 구간 구분
