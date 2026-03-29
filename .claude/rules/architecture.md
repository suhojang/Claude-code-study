---
description: 헥사고날 아키텍처 핵심 원칙. 모든 Java 소스 코드 작성 시 적용.
globs: src/main/java/**/*.java
---

# 헥사고날 아키텍처 원칙

## 계층 구조와 의존성 방향

```
[Adapter In] → [Port In] → [Domain Service] → [Port Out] ← [Adapter Out]
```

의존성은 반드시 **바깥에서 안쪽(Adapter → Domain)**으로만 흐른다.
Domain은 어떤 Adapter도 알지 못한다.

## Domain 계층 (`{module}/domain/`)

### domain/model/
- 순수 Java 클래스만 허용 (POJO)
- Spring, JPA, Jackson, Lombok 등 프레임워크 어노테이션 전면 금지
- Java 25 record를 Value Object에 적극 활용
- Entity는 일반 class로 작성하되 불변성 최대한 보장
- 모든 상태 변경은 의미 있는 도메인 메서드를 통해서만 수행
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
        // 생성 시점 검증
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
        // 도메인 이벤트 등록
        registerEvent(OrderCancelled.of(this.id, reason));
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

### domain/port/in/ (Inbound Port)
- Use Case 인터페이스 정의
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

### domain/port/out/ (Outbound Port)
- 외부 시스템과의 계약을 Domain이 정의
- 인터페이스 네이밍: `Save{Domain}Port`, `Load{Domain}Port`, `{Domain}EventPort`
- Domain이 필요로 하는 메서드만 정의 (ISP 원칙)
- 구현은 Adapter Out에서 담당

```java
public interface LoadOrderPort {
    Optional<Order> findById(OrderId id);
    List<Order> findByMemberId(MemberId memberId, PageRequest pageRequest);
}

public interface SaveOrderPort {
    OrderId save(Order order);
}
```

### domain/service/
- Inbound Port 구현체
- `@UseCase` 커스텀 어노테이션 부착 (내부적으로 `@Service` + `@Transactional`)
- Outbound Port만 주입받아 사용 (구현체 직접 참조 금지)
- 하나의 Service 클래스는 하나의 UseCase만 구현 (SRP)

```java
@UseCase
@RequiredArgsConstructor
public class CreateOrderService implements CreateOrderUseCase {

    private final LoadProductPort loadProductPort;
    private final SaveOrderPort saveOrderPort;
    private final OrderEventPort orderEventPort;

    @Override
    public OrderId execute(CreateOrderCommand command) {
        // 1. 상품 검증 (다른 모듈의 Outbound Port 사용)
        // 2. 주문 생성 (도메인 로직)
        // 3. 저장
        // 4. 이벤트 발행
    }
}
```

## Adapter 계층 (`{module}/adapter/`)

### adapter/in/web/ (Inbound Adapter - REST)
- Controller는 Inbound Port(UseCase)만 호출
- 비즈니스 로직 절대 금지 — 요청 변환 → UseCase 호출 → 응답 변환만 수행
- Request DTO → Command 변환은 Controller 또는 Request DTO 내부에서
- Domain Model → Response DTO 변환은 Response DTO의 정적 팩토리에서

### adapter/in/event/ (Inbound Adapter - Event)
- 다른 모듈이 발행한 ApplicationEvent를 수신
- `@EventListener` 또는 `@TransactionalEventListener` 사용
- 이벤트 수신 후 자기 모듈의 UseCase를 호출하여 처리

### adapter/out/persistence/ (Outbound Adapter - DB)
- Outbound Port 구현체: `{Domain}PersistenceAdapter`
- JPA Entity: `{Domain}JpaEntity` (domain model과 완전 분리)
- Mapper: `{Domain}PersistenceMapper` (MapStruct, domain ↔ JPA Entity 변환)
- Spring Data Repository: `{Domain}JpaRepository` (package-private)

### adapter/out/event/ (Outbound Adapter - Event)
- `OrderEventPort` 구현체
- `ApplicationEventPublisher`를 사용하여 도메인 이벤트를 Spring Event로 발행
- 모듈 간 느슨한 결합의 핵심 연결 고리

### adapter/out/external/ (Outbound Adapter - External API)
- PG사, 배송 추적, 알림 서비스 등 외부 API 클라이언트
- `RestClient` (Spring Boot 4.x 표준) 사용
- Circuit Breaker (Resilience4j) 적용 필수
