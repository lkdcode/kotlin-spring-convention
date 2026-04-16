# framework/security/filter — 필터 체인

## JwtAuthenticationFilter

매 요청마다 JWT 유효성 검증 후 SecurityContext 설정.

```kotlin
@Component
class JwtAuthenticationFilter(
    private val jwtService: JwtService,
    private val userDetailsService: UserDetailsService,
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val token = request.getHeader(TOKEN_HEADER_KEY)

        if (token != null && token.startsWith(TOKEN_PREFIX) && jwtService.isValid(token)) {
            val username = jwtService.getUsername(token)
            val userDetails = userDetailsService.loadUserByUsername(username)
            val authentication = UsernamePasswordAuthenticationToken(
                userDetails, null, userDetails.authorities,
            )
            SecurityContextHolder.getContext().authentication = authentication
        }

        filterChain.doFilter(request, response)
    }

    override fun shouldNotFilterAsyncDispatch() = false
    override fun shouldNotFilterErrorDispatch() = false
}
```

## JwtLoginFilter

로그인 요청 처리. JSON 바디에서 loginId/password 추출.

```kotlin
class JwtLoginFilter(
    private val jwtService: JwtService,
    private val userDetailsServiceAdapter: UserDetailsServiceAdapter,
    private val objectMapper: ObjectMapper,
) : UsernamePasswordAuthenticationFilter() {

    override fun attemptAuthentication(
        request: HttpServletRequest,
        response: HttpServletResponse,
    ): Authentication {
        val loginDTO = objectMapper.readValue(request.inputStream, LoginDTO::class.java)
        return userDetailsServiceAdapter
            .loadUserByUsername(loginDTO.loginId)
            ?.let { checkCredentialsAndGet(it, loginDTO.password, request) }
            ?: throw BadCredentialsException(FAIL_AUTH)
    }

    private fun checkCredentialsAndGet(
        userDetails: UserDetails,
        rawPassword: String,
        request: HttpServletRequest,
    ): Authentication {
        userDetailsServiceAdapter.recordLoginAttempt(userDetails.username, request)
        checkAttemptCount(userDetails)

        if (!passwordEncoder.matches(rawPassword, userDetails.password))
            throw BadCredentialsException(FAIL_AUTH)

        return UsernamePasswordAuthenticationToken(userDetails, null, userDetails.authorities)
    }

    override fun successfulAuthentication(
        request: HttpServletRequest,
        response: HttpServletResponse,
        chain: FilterChain,
        auth: Authentication,
    ) {
        val userDetails = auth.principal as AuthenticationDTO
        val successDTO = userDetailsServiceAdapter.successDTO(userDetails.username)
        val token = jwtService.create(userDetails.username, userDetails.role)

        response.setHeader(TOKEN_HEADER_KEY, token)
        userDetailsServiceAdapter.saveUserInformation(userDetails, request)
        ApiResponseWriter.write(response, ApiResponseCode.OK, successDTO)
    }

    override fun unsuccessfulAuthentication(
        request: HttpServletRequest,
        response: HttpServletResponse,
        failed: AuthenticationException,
    ) {
        ApiResponseWriter.write(response, ApiResponseCode.UNAUTHORIZED, failed.message)
    }
}
```

## ApiSecurityFilter (설정)

필터 체인 및 URL 권한 설정.

```kotlin
@Configuration
class ApiSecurityFilter(
    private val baseSecurity: BaseSecurity,
    private val jwtLoginFilter: JwtLoginFilter,
    private val jwtAuthenticationFilter: JwtAuthenticationFilter,
    private val jwtLogoutHandler: JwtLogoutHandler,
    private val jwtLogoutSuccessHandler: JwtLogoutSuccessHandler,
) {
    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain =
        http
            .with(baseSecurity) {}
            .addFilterAt(jwtLoginFilter, UsernamePasswordAuthenticationFilter::class.java)
            .addFilterAfter(jwtAuthenticationFilter, ExceptionTranslationFilter::class.java)
            .logout {
                it.logoutUrl("/logout")
                it.addLogoutHandler(jwtLogoutHandler)
                it.logoutSuccessHandler(jwtLogoutSuccessHandler)
            }
            .authorizeHttpRequests {
                it.requestMatchers("/login", "/logout", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                it.requestMatchers("/api/**", "/v2/api/**").authenticated()
                it.anyRequest().denyAll()
            }
            .build()
}
```

## JwtLogoutHandler / JwtLogoutSuccessHandler

```kotlin
@Component
class JwtLogoutHandler(
    private val jwtService: JwtService,
) : LogoutHandler {
    override fun logout(request: HttpServletRequest, response: HttpServletResponse, authentication: Authentication?) {
        request.getHeader(TOKEN_HEADER_KEY)?.let { jwtService.remove(it) }
    }
}

@Component
class JwtLogoutSuccessHandler : LogoutSuccessHandler {
    override fun onLogoutSuccess(request: HttpServletRequest, response: HttpServletResponse, authentication: Authentication?) {
        ApiResponseWriter.write(response, ApiResponseCode.OK, null)
    }
}
```

## BaseSecurity

```kotlin
@Component
class BaseSecurity(
    private val apiAuthenticationEntryPoint: ApiAuthenticationEntryPoint,
    private val apiAccessDeniedHandler: ApiAccessDeniedHandler,
    private val corsConfigurationSource: CorsConfigurationSource,
) : AbstractHttpConfigurer<BaseSecurity, HttpSecurity>() {

    override fun init(http: HttpSecurity) {
        http
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .csrf { it.disable() }
            .formLogin { it.disable() }
            .cors { it.configurationSource(corsConfigurationSource) }
            .exceptionHandling {
                it.authenticationEntryPoint(apiAuthenticationEntryPoint)
                it.accessDeniedHandler(apiAccessDeniedHandler)
            }
    }
}
```
