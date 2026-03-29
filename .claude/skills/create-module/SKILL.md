---
name: create-module
description: Spring Modulith + Hexagonal 아키텍처에 맞는 새 도메인 모듈의 전체 패키지 구조와 필수 파일을 스캐폴딩합니다
allowed-tools: Bash, Write, Read, Glob, Grep
---

## 새 도메인 모듈 생성

`$ARGUMENTS` 로 전달된 모듈명과 테이블 접두어로 아래 작업을 수행하세요.
인자 형식: `<모듈명> <테이블접두어>` (예: `inventory inv`)

### 1단계: 패키지 구조 생성

`src/main/java/com/example/app/{module}/` 아래에 다음 구조를 생성합니다:

```
{module}/
├── {Module}ModuleConfig.java          # @Configuration + SecurityCustomizer Bean
├── package-info.java                  # @ApplicationModule(allowedDependencies = {"_shared"})
├── api/                               # 공개 API (이벤트, 공개 인터페이스)
├── domain/
│   ├── model/                         # 순수 도메인 모델 (프레임워크 어노테이션 금지)
│   ├── port/in/                       # Inbound Port (UseCase, Query)
│   ├── port/out/                      # Outbound Port (Save, Load, Event)
│   ├── service/                       # UseCase 구현체 (@UseCase)
│   └── exception/                     # 도메인 예외 + ErrorCode enum
└── adapter/
    ├── in/
    │   ├── web/                       # REST Controller + Request/Response DTO
    │   └── event/                     # 외부 모듈 이벤트 수신 핸들러
    └── out/
        ├── persistence/               # JPA Entity, Repository, Mapper, Adapter
        └── event/                     # 이벤트 발행 Adapter
```

### 2단계: 필수 파일 생성

#### package-info.java
```java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {"_shared"}
)
package com.example.app.{module};
```

#### {Module}ModuleConfig.java
```java
@Configuration
class {Module}ModuleConfig {

    @Bean
    SecurityCustomizer {module}Security() {
        return http -> http.authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/v1/{modules}/**").hasRole("USER")
            .requestMatchers(HttpMethod.POST, "/api/v1/{modules}/**").hasRole("USER")
        );
    }
}
```

#### {Module}ErrorCode.java (domain/exception/)
```java
@Getter
@RequiredArgsConstructor
public enum {Module}ErrorCode implements ErrorCode {

    {MODULE}_NOT_FOUND("{PREFIX}-001", "{모듈명}을(를) 찾을 수 없습니다", 404);

    private final String code;
    private final String message;
    private final int status;
}
```

### 3단계: Flyway 마이그레이션 디렉토리

`src/main/resources/db/migration/{module}/` 디렉토리를 생성합니다.

### 4단계: 테스트 디렉토리

`src/test/java/com/example/app/{module}/` 아래에 도메인 테스트 구조를 생성합니다:
```
{module}/
├── domain/service/                    # 순수 단위 테스트
├── adapter/in/web/                    # @WebMvcTest
├── adapter/out/persistence/           # @DataJpaTest + Testcontainers
└── fixture/                           # {Module}Fixture.java
```

### 5단계: Rules 파일 생성

`.claude/rules/domain-{module}.md` 파일을 기존 도메인 규칙 파일 형식에 맞춰 생성합니다.

### 규칙
- Domain model에 Spring, JPA, Lombok 어노테이션 절대 금지
- JPA Entity 클래스명: `{Domain}JpaEntity`
- 테이블명: `{접두어}_{테이블명}` (snake_case)
- Repository는 package-private
- 다른 모듈 테이블에 FK 금지
