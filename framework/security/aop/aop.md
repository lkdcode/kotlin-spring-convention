# framework/security/aop — @CheckRole 권한 제어

API 메서드 단위 권한 검사. Security Config 의 URL 기반 접근 제어와 병행 사용.

## @CheckRole (어노테이션)

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class CheckRole(
    val authorities: Array<String>,
)
```

## PermissionAspect (AOP)

```kotlin
@Aspect
@Component
class PermissionAspect {

    @Around("@annotation(checkRole)")
    fun checkPermission(
        joinPoint: ProceedingJoinPoint,
        checkRole: CheckRole,
    ): Any? {
        val authentication = SecurityContextHolder.getContext().authentication
        val principal = authentication?.principal as? AuthenticationDTO
            ?: throw ApiException(ApiResponseCode.UNAUTHORIZED)

        val hasPermission = checkRole.authorities.any { it == principal.role }
        if (!hasPermission) throw ApiException(ApiResponseCode.FORBIDDEN)

        return joinPoint.proceed()
    }
}
```

## 사용 예시

```kotlin
@Operation(summary = "관리자 전용 API")
@CheckRole(authorities = ["ADMIN"])
@PostMapping("/api/admin/foos")
fun createAdmin(
    @Valid @RequestBody request: CreateFooRequest,
): ApiResponse<String> {
    createUsecase.execute(request.toModel())
    return ApiResponseCreated("생성에 성공했습니다.")
}

// 여러 역할 허용
@CheckRole(authorities = ["ADMIN", "MANAGER"])
@GetMapping("/api/foos")
fun fetchList(...): ApiResponse<FetchFooDTOList>
```

## Security Config URL 접근 제어 vs @CheckRole

| | Security Config | @CheckRole |
|---|---|---|
| 제어 단위 | URL 패턴 | 메서드 |
| 사용 시점 | 모든 요청 전 | 컨트롤러 진입 전 |
| 용도 | 인증 여부, 큰 단위 역할 구분 | 세밀한 역할/권한 제어 |
| 예시 | `/api/**` → authenticated | `ADMIN` 만 허용 |

## 규칙

- URL 기반 제어로 충분하면 `@CheckRole` 생략
- 같은 URL 에서 역할별로 다른 동작이 필요할 때 `@CheckRole` 사용
- `@CheckRole` 없는 메서드는 Security Config 의 URL 규칙만 적용
