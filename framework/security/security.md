# framework/security — Spring Security 컨벤션

JWT 기반 Stateless 인증. 세션 없음.

## 디렉토리 구조

```
framework/security/
├── security.md          ← 전체 개요 (이 파일)
├── jwt/jwt.md           ← JWT 컴포넌트
├── filter/filter.md     ← 필터 체인
├── auth/auth.md         ← 인증 객체
├── handler/handler.md   ← 예외 핸들러
└── aop/aop.md           ← @CheckRole AOP
```

---

## 전체 인증 흐름

```
로그인 (POST /login)
    → JwtLoginFilter
        → UserDetailsServiceAdapter.loadUserByUsername
        → BCrypt 비밀번호 검증
        → JwtService.create → Authorization 헤더 반환

요청 (GET /api/...)
    → JwtAuthenticationFilter
        → JwtService.isValid (블랙리스트 + 만료 확인)
        → SecurityContextHolder 저장
    → Controller
        → @CheckRole (AOP) → PermissionAspect

로그아웃 (POST /logout)
    → JwtLogoutHandler
        → JwtService.remove → 블랙리스트 추가
```

## 필터 체인 순서

```
JwtLoginFilter              ← UsernamePasswordAuthenticationFilter 위치
    ↓
[Spring Security 기본 필터]
    ↓
JwtAuthenticationFilter     ← ExceptionTranslationFilter 이후
    ↓
ApiAccessDeniedHandler / ApiAuthenticationEntryPoint
```

## 핵심 설계 원칙

- **Stateless** — 세션 없음, JWT 로 모든 인증 정보 전달
- **블랙리스트** — 로그아웃 토큰은 Cache 에 저장, 매 요청 시 확인
- **단일 책임** — JWT 기능을 Creator / Parser / Validator / Remover / Service 로 분리
- **통일된 에러 응답** — 401/403 모두 ApiResponseWriter 로 JSON 반환
