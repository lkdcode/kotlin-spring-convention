# framework/security/jwt — JWT (Access + Refresh Token)

## 토큰 구조

| 토큰 | 유효 시간 | 용도 |
|---|---|---|
| AccessToken | 30분 (1_800s) | API 요청 인증 |
| RefreshToken | 7일 (604_800s) | AccessToken 재발급 |

---

## 설정 파일 구조

```
resources/
├── dev/
│   └── secret/
│       └── jwt/
│           └── jwt-config.yml
└── prod/
    └── secret/
        └── jwt/
            └── jwt-config.yml
```

```yaml
# resources/{env}/secret/jwt/jwt-config.yml
jwt:
  secret-key: base64-encoded-secret-key-here
  issuer: your-service-name
  access:
    expired: 1_800       # 30분 (초 단위)
  refresh:
    expired: 604_800     # 7일 (초 단위)
```

---

## JwtValues

```kotlin
object JwtValues {
    const val TOKEN_HEADER_KEY = "Authorization"
    const val REFRESH_COOKIE_KEY = "Refresh-Token"
    const val TOKEN_PREFIX = "Bearer "
    const val BLACK_LIST_KEY = "BLACK_LIST"
    const val USERNAME_KEY = "username"
    const val USER_ROLE_KEY = "role"
    const val TOKEN_TYPE_KEY = "type"
    const val ACCESS_TOKEN = "ACCESS"
    const val REFRESH_TOKEN = "REFRESH"
}
```

## JwtProperties

```kotlin
@Component
data class JwtProperties(
    @Value("\${jwt.issuer}")
    val issuer: String,

    @Value("\${jwt.access.expired}")
    val accessExpired: Long,

    @Value("\${jwt.refresh.expired}")
    val refreshExpired: Long,

    @Value("\${jwt.secret-key}")
    private val secretKey: String,
) {
    fun getSecretKey(): SecretKey =
        Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey))
}
```

## JwtCreator

```kotlin
@Component
class JwtCreator(
    private val jwtProperties: JwtProperties,
) {
    fun createAccess(
        username: String,
        role: String,
    ): String = create(username, role, ACCESS_TOKEN, jwtProperties.accessExpired)

    fun createRefresh(
        username: String,
        role: String,
    ): String = create(username, role, REFRESH_TOKEN, jwtProperties.refreshExpired)

    private fun create(
        username: String,
        role: String,
        tokenType: String,
        expired: Long,
    ): String {
        val now = Date()
        return TOKEN_PREFIX + Jwts.builder()
            .claim(USERNAME_KEY, username)
            .claim(USER_ROLE_KEY, role)
            .claim(TOKEN_TYPE_KEY, tokenType)
            .issuedAt(now)
            .expiration(Date(now.time + expired * 1000))
            .signWith(jwtProperties.getSecretKey())
            .compact()
    }
}
```

## JwtParser

```kotlin
@Component
class JwtParser(
    private val jwtProperties: JwtProperties,
) {
    fun removePrefix(token: String): String =
        token.removePrefix(TOKEN_PREFIX)

    fun getClaims(token: String): Claims =
        Jwts.parser()
            .verifyWith(jwtProperties.getSecretKey())
            .build()
            .parseSignedClaims(removePrefix(token))
            .payload

    fun getUsername(token: String): String =
        getClaims(token)[USERNAME_KEY] as String

    fun getTokenType(token: String): String =
        getClaims(token)[TOKEN_TYPE_KEY] as String

    fun isAccessToken(token: String): Boolean =
        runCatching { getTokenType(token) == ACCESS_TOKEN }.getOrDefault(false)

    fun isRefreshToken(token: String): Boolean =
        runCatching { getTokenType(token) == REFRESH_TOKEN }.getOrDefault(false)
}
```

## JwtValidator

```kotlin
@Component
class JwtValidator(
    private val jwtRemover: JwtRemover,
    private val jwtParser: JwtParser,
) {
    fun validate(token: String): Boolean =
        isNotRemoved(token) && isNotExpired(token)

    private fun isNotRemoved(token: String): Boolean =
        jwtRemover.get(token) == null

    private fun isNotExpired(token: String): Boolean =
        runCatching { jwtParser.getClaims(token) }.isSuccess
}
```

## JwtRemover

```kotlin
@Component
class JwtRemover(
    private val cacheManager: CacheManager,
) {
    fun remove(token: String) {
        cacheManager.getCache(BLACK_LIST_KEY)?.put(token, token)
    }

    fun get(token: String): String? =
        cacheManager.getCache(BLACK_LIST_KEY)?.get(token, String::class.java)
}
```

## JwtService (파사드)

```kotlin
@Service
class JwtService(
    private val jwtCreator: JwtCreator,
    private val jwtParser: JwtParser,
    private val jwtValidator: JwtValidator,
    private val jwtRemover: JwtRemover,
) {
    // 로그인 시 Access + Refresh 동시 발급
    fun createTokenPair(
        username: String,
        role: String,
    ): TokenPair =
        TokenPair(
            accessToken = jwtCreator.createAccess(username, role),
            refreshToken = jwtCreator.createRefresh(username, role),
        )

    // AccessToken 재발급 (RefreshToken 으로)
    fun refresh(refreshToken: String): String {
        require(jwtParser.isRefreshToken(refreshToken)) { "RefreshToken 이 아닙니다" }
        require(jwtValidator.validate(refreshToken)) { "유효하지 않은 RefreshToken" }
        val username = jwtParser.getUsername(refreshToken)
        val role = jwtParser.getClaims(refreshToken)[USER_ROLE_KEY] as String
        return jwtCreator.createAccess(username, role)
    }

    fun getUsername(token: String): String =
        jwtParser.getUsername(token)

    fun isValidAccess(token: String): Boolean =
        jwtValidator.validate(token) && jwtParser.isAccessToken(token)

    // 로그아웃: Access + Refresh 모두 블랙리스트
    fun removeAll(accessToken: String, refreshToken: String) {
        jwtRemover.remove(jwtParser.removePrefix(accessToken))
        jwtRemover.remove(jwtParser.removePrefix(refreshToken))
    }

    // RefreshToken 쿠키 maxAge 설정용
    val refreshExpiredSeconds: Long get() = jwtProperties.refreshExpired
}

data class TokenPair(
    val accessToken: String,
    val refreshToken: String,
)
```

---

## 토큰 전달 방식

| 토큰 | 전달 방식 | 비고 |
|---|---|---|
| AccessToken | **헤더** (`Authorization: Bearer ...`) | 모든 API 요청에 포함 |
| RefreshToken | **쿠키** (`Refresh-Token`, HttpOnly, Secure) | 특수한 경우 QueryString 허용 |

## 인증 흐름

```
로그인 성공
    → createTokenPair()
    → 응답 헤더: Authorization (AccessToken)
    → 응답 쿠키: Refresh-Token (RefreshToken, HttpOnly, Secure)

API 요청
    → Authorization 헤더의 AccessToken 검증 (isValidAccess)
    → AccessToken 만료 시 401 반환

AccessToken 재발급 (POST /refresh)
    → 쿠키의 RefreshToken 검증
    → 새 AccessToken 발급

로그아웃
    → 헤더에서 AccessToken, 쿠키에서 RefreshToken 추출
    → Access + Refresh 모두 블랙리스트 등록
    → RefreshToken 쿠키 삭제 (maxAge = 0)
```

---

## 규칙

- Claims 에 민감 정보 절대 금지 — `username`, `role`, `type` 만 포함
- AccessToken 유효 시간: **30분** (1_800s)
- RefreshToken 유효 시간: **7일** (604_800s)
- 블랙리스트 캐시 TTL = RefreshToken 만료 시간 기준으로 설정
- `JwtService` 를 통해서만 JWT 기능 호출 — 내부 컴포넌트 직접 호출 금지
- 로그아웃 시 AccessToken + RefreshToken **둘 다** 블랙리스트 등록
- AccessToken → **헤더** (`Authorization: Bearer ...`)
- RefreshToken → **쿠키** (`Refresh-Token`, HttpOnly, Secure) — 특수한 경우 QueryString 허용
- RefreshToken 쿠키 `maxAge` = `JwtProperties.refreshExpired` 와 동일하게 설정
