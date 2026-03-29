---
description: 공유 커널(_shared) 모듈 규칙. 공통 인프라 코드 작성 시 적용.
globs: src/main/java/**/_shared/**/*.java, src/test/java/**/_shared/**/*.java
---

# 공유 커널 (_shared Module)

**소유팀**: team-platform
**Spring Modulith**: Named Module (`@NamedInterface("shared")`)

## 역할

모든 도메인 모듈이 공통으로 사용하는 인프라 코드를 제공한다.
**비즈니스 로직은 절대 포함하지 않는다.**

## 모듈 구조

```
_shared/
├── package-info.java               # @NamedInterface("shared")
├── config/
│   ├── JpaConfig.java              # JPA Auditing, QueryDSL 설정
│   ├── SecurityConfig.java         # 중앙 보안 설정 (모듈별 Customizer 수집)
│   ├── SecurityCustomizer.java     # 각 모듈이 구현하는 보안 커스터마이저 인터페이스
│   ├── RedisConfig.java            # Redis 연결 설정
│   └── AsyncConfig.java            # 비동기 설정
├── error/
│   ├── DomainException.java        # 도메인 예외 최상위 추상 클래스
│   ├── EntityNotFoundException.java
│   ├── BusinessRuleException.java
│   ├── InvalidInputException.java
│   ├── ExternalServiceException.java
│   ├── ErrorCode.java              # 에러코드 인터페이스
│   ├── ErrorResponse.java          # 에러 응답 record
│   ├── FieldErrorDetail.java       # 필드 에러 상세 record
│   └── GlobalExceptionHandler.java # @RestControllerAdvice
├── common/
│   ├── ApiResponse.java            # 성공 응답 래퍼 record
│   ├── PageResponse.java           # 페이징 응답 record
│   ├── BaseTimeEntity.java         # @MappedSuperclass (createdAt, updatedAt)
│   └── UseCase.java                # 커스텀 어노테이션 (@Service + @Transactional)
└── auth/
    ├── JwtProvider.java            # JWT 토큰 생성/검증
    ├── JwtAuthenticationFilter.java
    └── MemberPrincipal.java        # @AuthenticationPrincipal 타입
```

## 변경 규칙 (IMPORTANT)

- **platform 팀만 이 패키지를 수정**할 수 있다
- 변경 시 **하위 호환성 필수** (기존 모듈이 깨지면 안 됨)
- 새로운 공통 코드 추가 시 **최소 2개 이상의 모듈에서 필요한 경우만** 추가
- 하나의 모듈에서만 필요한 코드는 해당 모듈 내부에 배치

## 특정 도메인 모듈에 의존 금지

```java
// ❌ 절대 금지: _shared에서 특정 도메인 모듈 import
import com.example.app.order.domain.model.Order;     // ❌
import com.example.app.product.api.FindProductQuery;  // ❌

// ✅ _shared는 Java 표준 + Spring Framework만 의존
import java.time.LocalDateTime;
import org.springframework.stereotype.Service;
```

## 테스트 실행

```bash
./gradlew test --tests "*._shared.*"
```
