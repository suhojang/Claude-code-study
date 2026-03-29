# CLAUDE.md - 쇼핑몰 플랫폼 (Spring Boot 4.x Monolith + Hexagonal Architecture)

## 프로젝트 개요

Spring Boot 4.x 기반 모놀리스 쇼핑몰 플랫폼.
헥사고날 아키텍처(Ports & Adapters)를 적용하여 도메인 순수성을 보장하고,
모듈 경계를 엄격히 관리하여 향후 마이크로서비스 전환이 가능하도록 설계한다.

- **Language**: Java 21 (record, sealed class, pattern matching, virtual thread)
- **Framework**: Spring Boot 4.x, Spring Security 7.x
- **Build**: Gradle 8.x (Kotlin DSL)
- **Database**: PostgreSQL 16 + Redis 7 (캐시/세션)
- **Migration**: Flyway
- **Mapper**: MapStruct 1.6+
- **Test**: JUnit 5, Mockito, AssertJ, Testcontainers, ArchUnit

## 빌드 & 실행

```bash
# 빌드 (린트 + 테스트 + 컴파일)
./gradlew build

# 로컬 실행 (Docker Compose로 PostgreSQL, Redis 자동 구동)
./gradlew bootRun --args='--spring.profiles.active=local'

# 포맷팅
./gradlew spotlessApply

# 린트 검사
./gradlew spotlessCheck
```

## 테스트 명령어

```bash
# 전체 테스트
./gradlew test

# 도메인별 테스트
./gradlew test --tests "*.order.*"
./gradlew test --tests "*.product.*"

# 단일 테스트 클래스
./gradlew test --tests "com.example.app.order.domain.service.CreateOrderServiceTest"

# 통합 테스트 (Testcontainers - Docker 필요)
./gradlew integrationTest

# 아키텍처 테스트 (ArchUnit)
./gradlew test --tests "*.architecture.*"
```

## 패키지 구조

```
src/main/java/com/example/app/
├── order/                          # 주문 도메인 모듈
│   ├── domain/
│   │   ├── model/                  # Order, OrderItem, OrderStatus(enum)
│   │   ├── port/in/               # CreateOrderUseCase, FindOrderQuery, CancelOrderUseCase
│   │   ├── port/out/              # SaveOrderPort, LoadOrderPort, OrderEventPort
│   │   ├── service/               # CreateOrderService, FindOrderService, CancelOrderService
│   │   └── exception/             # OrderNotFoundException, OrderAlreadyCancelledException
│   └── adapter/
│       ├── in/web/                # OrderController, CreateOrderRequest, OrderResponse
│       ├── in/event/              # OrderEventListener (다른 모듈 이벤트 수신)
│       └── out/
│           ├── persistence/       # OrderPersistenceAdapter, OrderJpaEntity, OrderMapper
│           └── event/             # OrderSpringEventAdapter (ApplicationEvent 발행)
│
├── product/                        # 상품 도메인 모듈
├── member/                         # 회원 도메인 모듈
├── payment/                        # 결제 도메인 모듈
├── delivery/                       # 배송 도메인 모듈
│
└── global/                         # 전역 인프라 (특정 도메인 의존 금지)
    ├── config/                     # JpaConfig, SecurityConfig, RedisConfig, AsyncConfig
    ├── error/                      # GlobalExceptionHandler, ErrorResponse, ErrorCode
    ├── common/                     # ApiResponse<T>, PageResponse<T>, BaseTimeEntity
    ├── auth/                       # JwtProvider, AuthenticationFilter
    └── aop/                        # LoggingAspect, DistributedLockAspect
```

## 아키텍처 & 모듈 규칙

@.claude/rules/architecture.md
@.claude/rules/module-boundary.md

## 코딩 컨벤션

@.claude/rules/coding-conventions.md

## 테스트 전략

@.claude/rules/testing.md

## API 설계

@.claude/rules/api-design.md

## 영속성 계층

@.claude/rules/persistence.md

## 에러 처리

@.claude/rules/error-handling.md

## 보안

@.claude/rules/security.md

## 성능 & 최적화

@.claude/rules/performance.md

## DB 마이그레이션

- Flyway 경로: `src/main/resources/db/migration/`
- 파일명: `V{yyyyMMdd}{nn}__{description}.sql` (예: `V2026032901__create_order_tables.sql`)
- 테이블 접두어로 모듈 구분: `ord_`, `prd_`, `mbr_`, `pay_`, `dlv_`
- DDL 변경은 반드시 새 마이그레이션 파일 생성 (기존 파일 수정 금지)

## Git 워크플로우

- 브랜치: `feature/{ticket}-{short-desc}`, `fix/{ticket}-{short-desc}`, `refactor/{ticket}-{short-desc}`
- 커밋 메시지: Conventional Commits
  - `feat(order): 주문 취소 기능 추가`
  - `fix(payment): 결제 금액 계산 오류 수정`
  - `refactor(product): 상품 조회 쿼리 최적화`
  - `test(member): 회원 가입 서비스 단위 테스트 추가`
- PR 생성 전 `./gradlew build` 통과 필수
- PR 제목은 커밋 메시지와 동일한 Conventional Commits 형식

## 주의사항

- MapStruct 매퍼 인터페이스 변경 후 반드시 `./gradlew clean build` (annotation processor 캐시)
- Testcontainers 사용 시 Docker Desktop/Engine 실행 상태 확인
- `global/` 패키지에 비즈니스 로직을 절대 넣지 말 것
- 모듈 간 순환 참조 발생 시 반드시 ApplicationEvent로 분리
- `application.yml`만 사용 (`.properties` 금지)
- 환경별 설정: `application-{profile}.yml` (local, dev, staging, prod)
