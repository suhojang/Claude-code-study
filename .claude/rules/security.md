---
description: Spring Security 및 보안 규칙. 인증/인가 및 보안 관련 코드 작성 시 적용.
globs: src/main/java/**/global/auth/**/*.java, src/main/java/**/global/config/Security*.java, src/main/java/**/adapter/in/web/**/*.java
---

# 보안 규칙

## Spring Security 7.x 설정

```java
@Configuration
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/products/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

## 인증 (Authentication)

- JWT (Access Token + Refresh Token) 기반 Stateless 인증
- Access Token: 30분, Refresh Token: 7일
- 토큰 저장: Access Token은 클라이언트, Refresh Token은 Redis
- 토큰 검증: `JwtAuthenticationFilter`에서 `OncePerRequestFilter` 상속

## 인가 (Authorization)

### Controller 레벨 인가

```java
@PostMapping("/{orderId}/cancel")
@PreAuthorize("hasRole('USER') and @orderAuthorizationChecker.isOwner(#orderId, principal)")
ApiResponse<Void> cancelOrder(@PathVariable Long orderId) {
    // ...
}
```

### 규칙
- 역할(Role): `ROLE_USER`, `ROLE_ADMIN`, `ROLE_SELLER`
- 메서드 보안: `@PreAuthorize`로 세밀한 인가 처리
- 리소스 소유권 검증: 커스텀 `AuthorizationChecker` Bean 사용
- Controller에서 인증 정보: `@AuthenticationPrincipal MemberPrincipal`

## 입력 검증 및 방어

### SQL Injection 방어
- 직접 SQL 문자열 조합 전면 금지
- Spring Data JPA의 `@Query` + 파라미터 바인딩(`:param`) 사용
- QueryDSL은 기본적으로 파라미터 바인딩 적용

```java
// ✅ 파라미터 바인딩
@Query("SELECT o FROM OrderJpaEntity o WHERE o.memberId = :memberId")
List<OrderJpaEntity> findByMemberId(@Param("memberId") Long memberId);

// ❌ 금지: 문자열 조합
@Query("SELECT o FROM OrderJpaEntity o WHERE o.memberId = " + memberId) // 절대 금지
```

### XSS 방어
- 응답 Content-Type: `application/json` (HTML 렌더링 없음)
- 사용자 입력 문자열은 저장 전 HTML 이스케이프 또는 허용 패턴 검증
- Response Header: `X-Content-Type-Options: nosniff`

### 요청 크기 제한
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
  codec:
    max-in-memory-size: 1MB
```

## 민감 정보 관리

- `application.yml`에 비밀번호, API 키 직접 기입 금지
- 환경 변수 또는 Spring Cloud Config / Vault 사용
- 로그에 비밀번호, 토큰, 카드번호 등 민감 정보 출력 금지
- 예외 메시지에 민감 정보 포함 금지

```yaml
# ✅ 환경 변수 참조
spring:
  datasource:
    password: ${DB_PASSWORD}

jwt:
  secret: ${JWT_SECRET}
```

## CORS 설정

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PATCH", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### CORS 규칙
- `allowedOrigins`에 `*` 사용 금지 (운영 환경)
- 허용 도메인은 환경별 설정 파일에서 관리
- `local` 프로파일에서만 `localhost:*` 허용
