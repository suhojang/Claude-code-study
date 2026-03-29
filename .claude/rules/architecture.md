---
description: Spring Modulith + 헥사고날 아키텍처 핵심 원칙. 모든 Java 소스 코드 작성 시 적용.
globs: src/main/java/**/*.java
---

# 아키텍처 원칙 (Spring Modulith + Hexagonal)

## 이중 아키텍처 전략

```
┌─────────────────────────────────────────────────────────┐
│  Spring Modulith (모듈 간 경계)                           │
│  ┌──────────────┐  Event  ┌──────────────┐              │
│  │ order/       │ ──────→ │ payment/     │              │
│  │  ┌────────┐  │         │  ┌────────┐  │              │
│  │  │Hexagonal│  │         │  │Hexagonal│  │              │
│  │  │내부구조 │  │         │  │내부구조 │  │              │
│  │  └────────┘  │         │  └────────┘  │              │
│  └──────────────┘         └──────────────┘              │
└─────────────────────────────────────────────────────────┘
```

- **Spring Modulith**: 모듈 간 경계와 통신 규칙을 강제 (팀 간 충돌 방지)
- **Hexagonal Architecture**: 각 모듈 내부의 계층 구조와 의존성 방향을 관리

## Spring Modulith 모듈 구조

### Application Module = 최상위 패키지

Spring Modulith는 `com.example.app` 바로 아래의 각 패키지를 하나의 **Application Module**로 인식한다.

```
com.example.app.order/       → order Application Module
com.example.app.product/     → product Application Module
com.example.app.payment/     → payment Application Module
com.example.app._shared/     → shared module (Named Module: open)
```

### 공개 API vs 내부 구현 (가시성 규칙)

Spring Modulith의 가시성 규칙은 팀 간 충돌을 구조적으로 차단하는 핵심 메커니즘이다.

```
order/
├── api/                    ← ✅ 공개 (다른 모듈에서 import 가능)
│   ├── OrderCreatedEvent.java
│   ├── OrderCompletedEvent.java
│   └── FindOrderQuery.java     (필요 시 공개 조회 인터페이스)
├── OrderModuleConfig.java  ← ✅ 공개 (모듈 루트 패키지)
├── domain/                 ← 🔒 내부 (다른 모듈에서 import 불가)
│   ├── model/
│   ├── port/in/
│   ├── port/out/
│   ├── service/
│   └── exception/
└── adapter/                ← 🔒 내부 (다른 모듈에서 import 불가)
    ├── in/web/
    ├── in/event/
    └── out/persistence/
```

**규칙**:
- 모듈 루트 패키지(`order/`)의 클래스 → 공개
- `api/` 서브패키지 → 공개 (이벤트, 공유 인터페이스)
- `domain/`, `adapter/` 서브패키지 → **내부 전용, 외부 접근 불가**
- Spring Modulith가 컴파일 타임 + 테스트 타임에 위반을 자동 감지

### package-info.java로 모듈 메타데이터 선언

```java
// order/package-info.java
@org.springframework.modulith.ApplicationModule(
    allowedDependencies = {"_shared"}    // 명시적으로 의존 허용 모듈 선언
)
package com.example.app.order;
```

```java
// _shared/package-info.java
@org.springframework.modulith.NamedInterface("shared")
package com.example.app._shared;
```

## 헥사고날 아키텍처 (모듈 내부 구조)

### 의존성 방향

```
[Adapter In] → [Port In] → [Domain Service] → [Port Out] ← [Adapter Out]
```

의존성은 반드시 **바깥에서 안쪽(Adapter → Domain)**으로만 흐른다.
Domain은 어떤 Adapter도 알지 못한다.

### domain/model/ — 순수 도메인 모델
- 순수 Java 클래스만 허용 (POJO)
- Spring, JPA, Jackson, Lombok 등 프레임워크 어노테이션 전면 금지
- Java 25 record를 Value Object에 적극 활용
- 생성은 반드시 정적 팩토리 메서드 사용 (`Order.create(...)`)
- 자기 자신의 유효성은 생성 시점에 스스로 검증 (Self-Validating)

```java
// ✅ 올바른 도메인 모델
public class Order {
    private final OrderId id;
    private final MemberId memberId;
    private OrderStatus status;
    private final List<OrderItem> items;
    private final Money totalAmount;

    public static Order create(MemberId memberId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new InvalidOrderException("주문 항목은 최소 1개 이상이어야 합니다");
        }
        Money total = items.stream()
                .map(OrderItem::calculateAmount)
                .reduce(Money.ZERO, Money::add);
        return new Order(OrderId.generate(), memberId, OrderStatus.CREATED, items, total);
    }

    public void cancel(String reason) {
        if (!this.status.isCancellable()) {
            throw new OrderAlreadyCancelledException(this.id);
        }
        this.status = OrderStatus.CANCELLED;
    }
}
```

```java
// ❌ 금지: 프레임워크 어노테이션이 침투한 도메인 모델
@Entity
@Getter @Setter
@NoArgsConstructor
public class Order {
    @Id @GeneratedValue
    private Long id;
}
```

### domain/port/in/ — Inbound Port (Use Case)
- 명령(CUD)과 조회(R)를 분리 (CQRS 패턴)
- 명령: `{Action}{Domain}UseCase` (예: `CreateOrderUseCase`)
- 조회: `{Domain}{Action}Query` (예: `OrderFindQuery`)
- 파라미터는 전용 Command/Query record로 정의

```java
public interface CreateOrderUseCase {
    OrderId execute(CreateOrderCommand command);

    record CreateOrderCommand(
        MemberId memberId,
        List<OrderItemCommand> items,
        ShippingAddress shippingAddress
    ) {
        public CreateOrderCommand {
            Objects.requireNonNull(memberId, "memberId는 필수입니다");
            if (items == null || items.isEmpty()) {
                throw new IllegalArgumentException("주문 항목은 최소 1개 이상이어야 합니다");
            }
        }
    }
}
```

### domain/port/out/ — Outbound Port
- 인터페이스 네이밍: `Save{Domain}Port`, `Load{Domain}Port`, `{Domain}EventPort`
- Domain이 필요로 하는 메서드만 정의 (ISP 원칙)

### domain/service/ — Use Case 구현체
- `@UseCase` 커스텀 어노테이션 부착 (내부적으로 `@Service` + `@Transactional`)
- Outbound Port만 주입받아 사용
- 하나의 Service 클래스는 하나의 UseCase만 구현 (SRP)

## Adapter 계층

### adapter/in/web/ — REST Controller
- Controller는 Inbound Port(UseCase)만 호출
- 비즈니스 로직 절대 금지 — 요청 변환 → UseCase 호출 → 응답 변환만 수행

### adapter/in/event/ — 이벤트 수신 (모듈 간 통신의 수신 측)
- 다른 모듈이 발행한 이벤트를 수신하여 자기 모듈의 UseCase를 호출
- `@ApplicationModuleListener` (Spring Modulith) 사용
- 이벤트 핸들러 클래스 네이밍: `{SourceDomain}{Event}Handler`

```java
@Component
@RequiredArgsConstructor
class PaymentCompletedEventHandler {

    private final CompleteOrderUseCase completeOrderUseCase;

    @ApplicationModuleListener
    void on(PaymentCompletedEvent event) {
        completeOrderUseCase.execute(
            new CompleteOrderCommand(OrderId.of(event.orderId()))
        );
    }
}
```

### adapter/out/persistence/ — DB Adapter
- JPA Entity와 Domain Model 완전 분리
- MapStruct로 양방향 변환
- Spring Data Repository는 package-private

### adapter/out/event/ — 이벤트 발행 (모듈 간 통신의 발행 측)
- `ApplicationEventPublisher`를 사용하여 `api/` 패키지의 이벤트를 발행
- 발행 이벤트는 반드시 `api/` 패키지에 위치 (공개 API)

```java
@Component
@RequiredArgsConstructor
class OrderEventPublisherAdapter implements OrderEventPort {

    private final ApplicationEventPublisher publisher;

    @Override
    public void publishOrderCreated(Order order) {
        publisher.publishEvent(new OrderCreatedEvent(
            order.getId().value(),
            order.getMemberId().value(),
            order.getTotalAmount().value(),
            Instant.now()
        ));
    }
}
```

### adapter/out/external/ — 외부 API 클라이언트
- `RestClient` (Spring Boot 4.x 표준) 사용
- Circuit Breaker (Resilience4j) 적용 필수
