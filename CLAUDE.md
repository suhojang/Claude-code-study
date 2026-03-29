# CLAUDE.md - 쇼핑몰 플랫폼 (Spring Boot 4.x + Spring Modulith + Hexagonal Architecture)

## 프로젝트 개요

Spring Boot 4.x 기반 모듈러 모놀리스 쇼핑몰 플랫폼.
**Spring Modulith**로 모듈 경계를 프레임워크 레벨에서 강제하고,
각 모듈 내부는 **헥사고날 아키텍처(Ports & Adapters)**로 도메인 순수성을 보장한다.
모듈 간 통신은 **Spring Modulith Events**(비동기 이벤트)로만 수행하여,
팀원 간 코드 충돌을 구조적으로 차단하고 향후 마이크로서비스 전환이 가능하도록 설계한다.

- **Language**: Java 25 (record, sealed class, pattern matching, virtual thread, structured concurrency, scoped value, stable value, flexible constructor, module import, primitive pattern)
- **Framework**: Spring Boot 4.x, Spring Modulith 1.x, Spring Security 7.x
- **Build**: Gradle 8.x (Kotlin DSL)
- **Database**: PostgreSQL 16 + Redis 7 (캐시/세션)
- **Migration**: Flyway
- **Mapper**: MapStruct 1.6+
- **Test**: JUnit 5, Mockito, AssertJ, Testcontainers, Spring Modulith Test

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

# 모듈별 격리 테스트 (Spring Modulith)
./gradlew test --tests "*.order.*"
./gradlew test --tests "*.product.*"

# 단일 테스트 클래스
./gradlew test --tests "com.example.app.order.domain.service.CreateOrderServiceTest"

# 모듈 구조 검증 테스트
./gradlew test --tests "*.ModularityTests"

# 통합 테스트 (Testcontainers - Docker 필요)
./gradlew integrationTest

# 모듈 구조 검증 테스트 (Spring Modulith)
./gradlew test --tests "*.ModularityTests"
```

## 패키지 구조 (Spring Modulith 모듈 = 최상위 패키지)

```
src/main/java/com/example/app/
│
├── order/                              # 📦 주문 Application Module
│   ├── OrderModuleConfig.java          #   모듈 전용 Spring 설정 (@Configuration)
│   ├── api/                            #   ✅ 공개 API (다른 모듈이 참조 가능한 유일한 패키지)
│   │   ├── OrderCreatedEvent.java      #     발행 이벤트 (record)
│   │   ├── OrderCompletedEvent.java    #     발행 이벤트 (record)
│   │   └── OrderCancelledEvent.java    #     발행 이벤트 (record)
│   ├── domain/                         #   🔒 내부 전용 (internal)
│   │   ├── model/                      #     Order, OrderItem, OrderStatus, Money
│   │   ├── port/in/                    #     CreateOrderUseCase, FindOrderQuery
│   │   ├── port/out/                   #     SaveOrderPort, LoadOrderPort
│   │   ├── service/                    #     CreateOrderService, CancelOrderService
│   │   └── exception/                  #     OrderNotFoundException
│   └── adapter/                        #   🔒 내부 전용 (internal)
│       ├── in/
│       │   ├── web/                    #     OrderController, DTO
│       │   └── event/                  #     PaymentCompletedEventHandler (외부 이벤트 수신)
│       └── out/
│           ├── persistence/            #     OrderJpaEntity, OrderRepository, OrderMapper
│           └── event/                  #     OrderEventPublisherAdapter
│
├── product/                            # 📦 상품 Application Module
│   ├── ProductModuleConfig.java
│   ├── api/                            #   공개 이벤트 + 공개 인터페이스
│   ├── domain/                         #   내부 전용
│   └── adapter/                        #   내부 전용
│
├── member/                             # 📦 회원 Application Module
├── payment/                            # 📦 결제 Application Module
├── delivery/                           # 📦 배송 Application Module
├── notification/                       # 📦 알림 Application Module
│
└── _shared/                            # 📦 공유 커널 (global 대체, 최소한으로 유지)
    ├── config/                         #   JpaConfig, SecurityConfig, RedisConfig
    ├── error/                          #   GlobalExceptionHandler, ErrorResponse, ErrorCode
    ├── common/                         #   ApiResponse<T>, PageResponse<T>, BaseTimeEntity
    └── auth/                           #   JwtProvider, AuthenticationFilter
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

## 도메인 모듈별 상세 규칙

`.claude/rules/domain-*.md`에 각 모듈의 상세 규칙이 정의되어 있다.
**globs 패턴으로 해당 모듈 파일 작업 시 자동으로 로드된다.**

- 주문: `.claude/rules/domain-order.md` → `**/order/**/*.java`
- 상품: `.claude/rules/domain-product.md` → `**/product/**/*.java`
- 회원: `.claude/rules/domain-member.md` → `**/member/**/*.java`
- 결제: `.claude/rules/domain-payment.md` → `**/payment/**/*.java`
- 배송: `.claude/rules/domain-delivery.md` → `**/delivery/**/*.java`
- 알림: `.claude/rules/domain-notification.md` → `**/notification/**/*.java`
- 공유: `.claude/rules/domain-shared.md` → `**/_shared/**/*.java`

- Flyway 경로: **모듈별 분리** `src/main/resources/db/migration/{module}/`
  - `db/migration/order/V2026032901__create_order_tables.sql`
  - `db/migration/product/V2026032901__create_product_tables.sql`
- 파일명: `V{yyyyMMdd}{nn}__{description}.sql`
- 테이블 접두어로 모듈 구분: `ord_`, `prd_`, `mbr_`, `pay_`, `dlv_`, `ntf_`
- DDL 변경은 반드시 새 마이그레이션 파일 생성 (기존 파일 수정 금지)
- 다른 모듈의 마이그레이션 파일 수정 금지 (소유 모듈 담당자만 변경)

## Git 워크플로우 & 협업 충돌 방지

- 브랜치: `feature/{ticket}-{short-desc}`, `fix/{ticket}-{short-desc}`, `refactor/{ticket}-{short-desc}`
- 커밋 메시지: Conventional Commits (scope는 모듈명)
  - `feat(order): 주문 취소 기능 추가`
  - `fix(payment): 결제 금액 계산 오류 수정`
  - `refactor(product): 상품 조회 쿼리 최적화`
  - `test(member): 회원 가입 서비스 단위 테스트 추가`
- PR 생성 전 `./gradlew build` 통과 필수
- PR 제목은 커밋 메시지와 동일한 Conventional Commits 형식
- **CODEOWNERS**: 각 모듈 디렉토리별 담당팀 지정하여 교차 수정 사전 리뷰 강제
- **하나의 PR은 하나의 모듈만 수정** 원칙 (cross-module PR은 반드시 리뷰어 2인 이상)

## 주의사항

- MapStruct 매퍼 인터페이스 변경 후 반드시 `./gradlew clean build` (annotation processor 캐시)
- Testcontainers 사용 시 Docker Desktop/Engine 실행 상태 확인
- `_shared/` 패키지에 비즈니스 로직을 절대 넣지 말 것
- 모듈 간 직접 메서드 호출 금지 — 반드시 이벤트 또는 공개 API 인터페이스를 통해서만 통신
- `application.yml`만 사용 (`.properties` 금지)
- 환경별 설정: `application-{profile}.yml` (local, dev, staging, prod)
- 다른 팀이 소유한 모듈의 `api/` 패키지 변경 시 반드시 해당 팀과 사전 협의
