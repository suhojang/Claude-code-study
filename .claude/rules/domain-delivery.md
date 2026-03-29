---
description: 배송 모듈 규칙. 배송 도메인 코드 작성 시 적용.
globs: src/main/java/**/delivery/**/*.java, src/test/java/**/delivery/**/*.java
---

# 배송 모듈 (Delivery Module)

**소유팀**: team-delivery
**테이블 접두어**: `dlv_`
**API 경로**: `/api/v1/deliveries`

## 모듈 구조

```
delivery/
├── DeliveryModuleConfig.java
├── api/                            # ✅ 공개 API
│   ├── DeliveryStartedEvent.java
│   ├── DeliveryCompletedEvent.java
│   └── FindDeliveryQuery.java      # 배송 조회 공개 인터페이스
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Delivery.java
│   │   ├── DeliveryId.java
│   │   ├── DeliveryStatus.java     # PREPARING, SHIPPED, IN_TRANSIT, DELIVERED
│   │   ├── TrackingNumber.java     # VO
│   │   └── ShippingAddress.java    # VO
│   ├── port/in/
│   │   ├── StartDeliveryUseCase.java
│   │   ├── UpdateTrackingUseCase.java
│   │   └── DeliveryFindQuery.java
│   ├── port/out/
│   │   ├── SaveDeliveryPort.java
│   │   ├── LoadDeliveryPort.java
│   │   ├── DeliveryEventPort.java
│   │   └── DeliveryTrackingPort.java  # 외부 배송 추적 API
│   ├── service/
│   │   ├── StartDeliveryService.java
│   │   ├── UpdateTrackingService.java
│   │   └── FindDeliveryService.java
│   └── exception/
│       ├── DeliveryErrorCode.java    # DLV-001 ~ DLV-xxx
│       └── DeliveryNotFoundException.java
└── adapter/
    ├── in/
    │   ├── web/
    │   │   ├── DeliveryController.java
    │   │   └── DeliveryResponse.java
    │   └── event/
    │       ├── OrderCompletedEventHandler.java    # 주문 확정 → 배송 시작
    │       └── OrderCancelledEventHandler.java    # 주문 취소 → 배송 취소
    └── out/
        ├── persistence/
        │   ├── DeliveryPersistenceAdapter.java
        │   ├── DeliveryJpaEntity.java
        │   ├── DeliveryJpaRepository.java
        │   └── DeliveryPersistenceMapper.java
        ├── event/
        │   └── DeliveryEventPublisherAdapter.java
        └── external/
            └── DeliveryTrackingAdapter.java       # 외부 배송 추적 API
```

## 발행/수신 이벤트

| 발행 이벤트 | 시점 | 소비 모듈 |
|:---|:---|:---|
| `DeliveryStartedEvent` | 배송 출발 | order (상태 변경), notification |
| `DeliveryCompletedEvent` | 배송 완료 | order (주문 완료 처리), notification |

| 수신 이벤트 | 발행 모듈 | 처리 내용 |
|:---|:---|:---|
| `OrderCompletedEvent` | order | 배송 준비 시작 |
| `OrderCancelledEvent` | order | 배송 취소 (SHIPPED 이전만) |

## 비즈니스 규칙

- 배송은 주문 확정(CONFIRMED) 이후에만 시작 가능
- 배송 추적 번호는 출발 시점에 발급
- 배송 완료 후 취소 불가

## 테스트 실행

```bash
./gradlew test --tests "*.delivery.*"
```
