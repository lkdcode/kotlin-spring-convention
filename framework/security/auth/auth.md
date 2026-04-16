# framework/security/auth — 인증 객체

## AuthenticationDTO (UserDetails 구현)

SecurityContext 에 저장되는 인증 객체.

```kotlin
data class AuthenticationDTO(
    val id: Long,
    val role: String,
    val loginId: String,
    val encodedPassword: String,
    val isNotDeleted: Boolean,
    val isNonLocked: Boolean,
) : UserDetails {

    override fun getUsername(): String = loginId
    override fun getPassword(): String = encodedPassword
    override fun getAuthorities(): Collection<GrantedAuthority> =
        listOf(SimpleGrantedAuthority(role))

    override fun isAccountNonExpired() = true
    override fun isAccountNonLocked() = isNonLocked
    override fun isCredentialsNonExpired() = true
    override fun isEnabled() = isNotDeleted
}
```

## UserDetailsServiceAdapter (UserDetailsService 구현)

```kotlin
@Service
class UserDetailsServiceAdapter(
    private val userQueryPort: UserQueryPort,
    private val loginAttemptCommandPort: LoginAttemptCommandPort,
) : UserDetailsService {

    override fun loadUserByUsername(username: String): AuthenticationDTO? =
        userQueryPort.findByLoginId(username)?.toAuthenticationDTO()

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun recordLoginAttempt(username: String, request: HttpServletRequest) {
        loginAttemptCommandPort.record(username, request.remoteAddr)
    }

    fun successDTO(username: String): LoginSuccessDTO =
        userQueryPort.getLoginSuccessInfo(username)

    fun saveUserInformation(userDetails: AuthenticationDTO, request: HttpServletRequest) {
        loginAttemptCommandPort.saveLastLogin(userDetails.id, request.remoteAddr)
    }
}
```

## SecurityContextElement (코루틴 지원)

코루틴에서도 SecurityContext 가 전파되도록 지원.

```kotlin
class SecurityContextElement(
    private val securityContext: SecurityContext = SecurityContextHolder.getContext(),
) : ThreadContextElement<SecurityContext?> {

    companion object Key : CoroutineContext.Key<SecurityContextElement>
    override val key: CoroutineContext.Key<*> get() = Key

    override fun updateThreadContext(context: CoroutineContext): SecurityContext? {
        val previous = SecurityContextHolder.getContext()
        SecurityContextHolder.setContext(securityContext)
        return previous
    }

    override fun restoreThreadContext(context: CoroutineContext, oldState: SecurityContext?) {
        if (oldState == null) SecurityContextHolder.clearContext()
        else SecurityContextHolder.setContext(oldState)
    }
}
```

## UserIdAuditorAware

JPA Auditing (`@CreatedBy`, `@LastModifiedBy`) 에 현재 로그인 사용자 ID 주입.

```kotlin
@Component
class UserIdAuditorAware : AuditorAware<Long> {
    override fun getCurrentAuditor(): Optional<Long> {
        val authentication = SecurityContextHolder.getContext().authentication
        if (authentication == null || !authentication.isAuthenticated) return Optional.of(0L)
        val principal = authentication.principal as? AuthenticationDTO ?: return Optional.of(0L)
        return Optional.of(principal.id)
    }
}
```

## 규칙

- `AuthenticationDTO` 에 민감 정보(비밀번호 제외) 최소화
- `loadUserByUsername` — 사용자 없으면 `null` 반환 (필터에서 처리)
- `recordLoginAttempt` — **`REQUIRES_NEW`** 트랜잭션 필수 (로그인 실패해도 기록 커밋)
- 코루틴 사용 시 `SecurityContextElement` 를 `CoroutineContext` 에 추가
