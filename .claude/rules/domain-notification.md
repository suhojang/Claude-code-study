---
description: 알림 모듈 규칙. 알림 도메인 코드 작성 시 적용.
globs: src/main/java/**/notification/**/*.java, src/test/java/**/notification/**/*.java
---

# 알림 모듈 (Notification Module)

**소유팀**: team-platform
**테이블 접두어**: `ntf_`
**API 경로**: `/api/v1/notifications`

## 모듈 구조

```
notification/
├── NotificationModuleConfig.java
├── api/                            # ✅ 공개 API (최소한)
│   └── (이 모듈은 주로 이벤트를 수신만 하므로 공개 이벤트 없음)
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Notification.java
│   │   ├── NotificationId.java
│   │   ├── NotificationType.java   # enum: EMAIL, SMS, PUSH
│   │   └── NotificationStatus.java # enum: PENDING, SENT, FAILED
│   ├── port/in/
│   │   ├── SendNotificationUseCase.java
│   │   └── NotificationFindQuery.java
│   ├── port/out/
│   │   ├── SaveNotificationPort.java
│   │   ├── LoadNotificationPort.java
│   │   ├── EmailSenderPort.java
│   │   └── SmsSenderPort.java
│   ├── service/
│   │   ├── SendNotificationService.java
│   │   └── FindNotificationService.java
│   └── exception/
│       ├── NotificationErrorCode.java  # NTF-001 ~ NTF-xxx
│       └── NotificationSendFailedException.java
└── adapter/
    ├── in/
    │   ├── web/
    │   │   └── NotificationController.java
    │   └── event/
    │       ├── OrderCreatedEventHandler.java       # 주문 접수 알림
    │       ├── PaymentCompletedEventHandler.java   # 결제 완료 알림
    │       ├── DeliveryStartedEventHandler.java    # 배송 시작 알림
    │       ├── DeliveryCompletedEventHandler.java  # 배송 완료 알림
    │       └── MemberRegisteredEventHandler.java   # 환영 메일
    └── out/
        ├── persistence/
        │   ├── NotificationPersistenceAdapter.java
        │   ├── NotificationJpaEntity.java
        │   └── NotificationJpaRepository.java
        └── external/
            ├── EmailSenderAdapter.java             # 이메일 발송 (SMTP/SES)
            └── SmsSenderAdapter.java               # SMS 발송 (외부 API)
```

## 수신 이벤트 (이 모듈은 이벤트 소비자)

| 수신 이벤트 | 발행 모듈 | 처리 내용 |
|:---|:---|:---|
| `OrderCreatedEvent` | order | 주문 접수 확인 알림 |
| `PaymentCompletedEvent` | payment | 결제 완료 알림 |
| `PaymentFailedEvent` | payment | 결제 실패 알림 |
| `DeliveryStartedEvent` | delivery | 배송 시작 알림 |
| `DeliveryCompletedEvent` | delivery | 배송 완료 알림 |
| `MemberRegisteredEvent` | member | 환영 메일 발송 |
| `RefundCompletedEvent` | payment | 환불 완료 알림 |

## 특징

- **이벤트 소비 전용 모듈**: 다른 모듈의 이벤트를 수신하여 알림 발송
- 공개 API 이벤트를 거의 발행하지 않음 (fire-and-forget)
- 알림 발송 실패 시 `@Retryable`로 재시도 (최대 3회)
- 재시도 초과 시 `ntf_notification` 테이블에 FAILED 상태로 기록

## 비즈니스 규칙

- 동일 대상에 동일 유형의 알림을 1분 내 중복 발송하지 않음
- 야간(22시~08시) SMS 발송 제한 (이메일은 허용)
- 알림 발송 실패는 비즈니스 크리티컬하지 않으므로 전체 트랜잭션에 영향 주지 않음

## 테스트 실행

```bash
./gradlew test --tests "*.notification.*"
```
