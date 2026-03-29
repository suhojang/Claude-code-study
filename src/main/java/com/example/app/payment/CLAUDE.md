# 결제 모듈 (Payment Module)

**소유팀**: team-payment
**테이블 접두어**: `pay_`
**API 경로**: `/api/v1/payments`

## 모듈 구조

```
payment/
├── PaymentModuleConfig.java
├── api/                            # ✅ 공개 API
│   ├── PaymentCompletedEvent.java
│   ├── PaymentFailedEvent.java
│   └── RefundCompletedEvent.java
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Payment.java
│   │   ├── PaymentId.java
│   │   ├── PaymentMethod.java      # enum: CARD, BANK_TRANSFER, VIRTUAL_ACCOUNT
│   │   ├── PaymentStatus.java      # enum: PENDING, COMPLETED, FAILED, REFUNDED
│   │   └── RefundReason.java
│   ├── port/in/
│   │   ├── CreatePaymentUseCase.java
│   │   ├── RefundPaymentUseCase.java
│   │   └── PaymentFindQuery.java
│   ├── port/out/
│   │   ├── SavePaymentPort.java
│   │   ├── LoadPaymentPort.java
│   │   ├── PaymentEventPort.java
│   │   └── PaymentGatewayPort.java  # 외부 PG사 API
│   ├── service/
│   │   ├── CreatePaymentService.java
│   │   ├── RefundPaymentService.java
│   │   └── FindPaymentService.java
│   └── exception/
│       ├── PaymentErrorCode.java     # PAY-001 ~ PAY-xxx
│       ├── PaymentNotFoundException.java
│       ├── PaymentAlreadyCompletedException.java
│       └── PaymentGatewayException.java
└── adapter/
    ├── in/
    │   ├── web/
    │   │   ├── PaymentController.java
    │   │   └── PaymentResponse.java
    │   └── event/
    │       ├── OrderCreatedEventHandler.java    # 주문 생성 → 결제 요청
    │       └── OrderCancelledEventHandler.java  # 주문 취소 → 환불 처리
    └── out/
        ├── persistence/
        │   ├── PaymentPersistenceAdapter.java
        │   ├── PaymentJpaEntity.java
        │   ├── PaymentJpaRepository.java
        │   └── PaymentPersistenceMapper.java
        ├── event/
        │   └── PaymentEventPublisherAdapter.java
        └── external/
            └── PgPaymentGatewayAdapter.java     # PG사 RestClient + CircuitBreaker
```

## 발행/수신 이벤트

| 발행 이벤트 | 시점 | 소비 모듈 |
|:---|:---|:---|
| `PaymentCompletedEvent` | PG 결제 승인 완료 | order (주문 확정), notification |
| `PaymentFailedEvent` | PG 결제 실패 | order (주문 실패 처리), notification |
| `RefundCompletedEvent` | 환불 처리 완료 | notification (환불 알림) |

| 수신 이벤트 | 발행 모듈 | 처리 내용 |
|:---|:---|:---|
| `OrderCreatedEvent` | order | 결제 요청 생성 + PG사 승인 요청 |
| `OrderCancelledEvent` | order | 결제 건 환불 처리 |

## 비즈니스 규칙

- 결제 금액은 주문 금액과 일치해야 함
- PG사 API 호출 시 Circuit Breaker 필수 (timeout 3초, retry 3회)
- 결제 완료 후 중복 결제 방지 (멱등성 키)
- 환불은 결제 완료 상태에서만 가능

## 외부 연동 (PG사)

- `RestClient` 사용
- `@CircuitBreaker` + `@Retry` + `@TimeLimiter` 적용 필수
- PG사 장애 시 `PaymentGatewayException` 발생, fallback으로 재시도 큐 등록

## 테스트 실행

```bash
./gradlew test --tests "*.payment.*"
```
