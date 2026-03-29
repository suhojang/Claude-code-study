---
description: 주문 모듈 규칙. 주문 도메인 코드 작성 시 적용.
globs: src/main/java/**/order/**/*.java, src/test/java/**/order/**/*.java
---

# 주문 모듈 (Order Module)

**소유팀**: team-order
**테이블 접두어**: `ord_`
**API 경로**: `/api/v1/orders`

## 모듈 구조

```
order/
├── OrderModuleConfig.java          # 모듈 설정 (SecurityCustomizer 포함)
├── api/                            # ✅ 공개 API (다른 모듈 참조 가능)
│   ├── OrderCreatedEvent.java      #   주문 생성 이벤트
│   ├── OrderCompletedEvent.java    #   주문 완료 이벤트
│   ├── OrderCancelledEvent.java    #   주문 취소 이벤트
│   └── FindOrderQuery.java         #   주문 조회 공개 인터페이스 (필요 시)
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Order.java              #   주문 엔티티 (도메인 모델)
│   │   ├── OrderItem.java          #   주문 항목 VO
│   │   ├── OrderId.java            #   주문 식별자 (record)
│   │   ├── OrderStatus.java        #   주문 상태 (enum/sealed)
│   │   └── Money.java              #   금액 VO (record)
│   ├── port/in/
│   │   ├── CreateOrderUseCase.java
│   │   ├── CancelOrderUseCase.java
│   │   └── OrderFindQuery.java     #   내부 조회 (Controller 전용)
│   ├── port/out/
│   │   ├── SaveOrderPort.java
│   │   ├── LoadOrderPort.java
│   │   └── OrderEventPort.java
│   ├── service/
│   │   ├── CreateOrderService.java #   1 Service = 1 UseCase
│   │   ├── CancelOrderService.java
│   │   └── FindOrderService.java
│   └── exception/
│       ├── OrderErrorCode.java     #   ORD-001 ~ ORD-xxx
│       ├── OrderNotFoundException.java
│       └── OrderAlreadyCancelledException.java
└── adapter/                        # 🔒 내부 전용
    ├── in/
    │   ├── web/
    │   │   ├── OrderController.java
    │   │   ├── CreateOrderRequest.java
    │   │   ├── CancelOrderRequest.java
    │   │   └── OrderResponse.java
    │   └── event/
    │       └── PaymentCompletedEventHandler.java   # payment 이벤트 수신
    └── out/
        ├── persistence/
        │   ├── OrderPersistenceAdapter.java
        │   ├── OrderJpaEntity.java
        │   ├── OrderItemJpaEntity.java
        │   ├── OrderJpaRepository.java             # package-private
        │   ├── OrderPersistenceMapper.java         # MapStruct
        │   └── OrderQueryRepository.java           # QueryDSL (복잡한 조회)
        └── event/
            └── OrderEventPublisherAdapter.java     # 이벤트 발행
```

## 발행 이벤트 (api/ 패키지)

이 모듈이 발행하는 이벤트. 다른 모듈은 이 이벤트만 구독 가능하다.

| 이벤트 | 발행 시점 | 소비 모듈 |
|:---|:---|:---|
| `OrderCreatedEvent` | 주문 생성 완료 | payment (결제 요청), notification (알림) |
| `OrderCompletedEvent` | 주문 확정 | delivery (배송 시작), notification (알림) |
| `OrderCancelledEvent` | 주문 취소 | payment (환불), delivery (배송 취소) |

## 수신 이벤트

이 모듈이 수신하는 다른 모듈의 이벤트.

| 이벤트 | 발행 모듈 | 처리 내용 |
|:---|:---|:---|
| `PaymentCompletedEvent` | payment | 주문 상태를 CONFIRMED로 변경 |
| `PaymentFailedEvent` | payment | 주문 상태를 PAYMENT_FAILED로 변경 |
| `DeliveryCompletedEvent` | delivery | 주문 상태를 DELIVERED로 변경 |

## 주문 상태 머신

```
CREATED → CONFIRMED → SHIPPED → DELIVERED → COMPLETED
   ↓          ↓
CANCELLED  CANCELLED (결제 전까지 취소 가능)
   ↓
PAYMENT_FAILED
```

## 비즈니스 규칙

- 주문 항목은 최소 1개 이상
- 주문 금액은 총 항목 금액의 합과 일치해야 함
- 취소는 SHIPPED 이전 상태에서만 가능
- 상품 재고 확인은 `product` 모듈의 `FindProductQuery` (공개 API)로 조회

## 테스트 실행

```bash
# 주문 모듈 전체 테스트
./gradlew test --tests "*.order.*"

# 도메인 서비스 단위 테스트
./gradlew test --tests "*.order.domain.service.*"

# 주문 모듈 격리 테스트 (Spring Modulith)
./gradlew test --tests "*.order.OrderModuleIntegrationTest"
```
