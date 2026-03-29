---
description: Java 25 + Spring Boot 4.x + Spring Modulith 코딩 컨벤션. 모든 Java 소스 코드에 적용.
globs: src/main/java/**/*.java
---

# 코딩 컨벤션

## Java 25 기능 활용

### Record
- DTO, Command, Query, Value Object, Event에 적극 사용
- record의 Compact Constructor로 입력값 검증 수행

```java
public record CreateOrderCommand(
    MemberId memberId,
    List<OrderItemCommand> items,
    ShippingAddress shippingAddress
) {
    public CreateOrderCommand {
        Objects.requireNonNull(memberId, "memberId는 필수입니다");
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("주문 항목은 최소 1개 이상이어야 합니다");
        }
        items = List.copyOf(items);  // 불변 방어 복사
    }
}
```

### Sealed Class / Interface
- 도메인 이벤트 계층 구조, 상태 머신에 활용

```java
public sealed interface OrderEvent permits OrderCreated, OrderCancelled, OrderCompleted {
    OrderId orderId();
    LocalDateTime occurredAt();
}

public record OrderCreated(OrderId orderId, MemberId memberId, LocalDateTime occurredAt)
    implements OrderEvent {}
```

### Pattern Matching
- `instanceof` 패턴 매칭, switch 표현식 적극 활용
- **Primitive Pattern (Java 25)**: switch에서 primitive 타입 직접 매칭

```java
return switch (event) {
    case OrderCreated e -> handleCreated(e);
    case OrderCancelled e -> handleCancelled(e);
    case OrderCompleted e -> handleCompleted(e);
};

// ✅ Primitive Pattern (Java 25)
return switch (statusCode) {
    case 0 -> "PENDING";
    case 1 -> "CONFIRMED";
    case int i when i < 0 -> "INVALID";
    case int i -> "UNKNOWN(" + i + ")";
};
```

### Module Import Declarations (Java 25)
- `import module` 구문으로 모듈 전체 패키지를 한 번에 import

```java
import module java.base;    // java.util.*, java.io.*, java.time.* 등 일괄 import
import com.example.app.order.domain.model.Order;  // 명시적 import 혼용 가능
```

### Flexible Constructor Bodies (Java 25)
- 생성자에서 `super()` / `this()` 호출 전에 검증 로직 배치 가능

```java
public class OrderItem {
    private final ProductId productId;
    private final int quantity;
    private final Money price;

    public OrderItem(ProductId productId, int quantity, Money price) {
        // ✅ Java 25: super() 전에 검증 가능
        if (quantity <= 0) {
            throw new InvalidOrderException("수량은 1 이상이어야 합니다");
        }
        Objects.requireNonNull(productId, "productId는 필수입니다");
        this.productId = productId;
        this.quantity = quantity;
        this.price = price;
    }
}
```

### Structured Concurrency (Java 25)
- `StructuredTaskScope`로 병렬 작업의 생명주기를 구조적으로 관리

```java
public OrderDetailInfo findOrderDetail(OrderId orderId) throws Exception {
    try (var scope = StructuredTaskScope.open()) {
        var orderTask = scope.fork(() -> loadOrderPort.findById(orderId));
        var paymentTask = scope.fork(() -> loadPaymentPort.findByOrderId(orderId));
        var deliveryTask = scope.fork(() -> loadDeliveryPort.findByOrderId(orderId));

        scope.join();

        return new OrderDetailInfo(
            orderTask.get(), paymentTask.get(), deliveryTask.get()
        );
    }
}
```

### Scoped Values (Java 25)
- `ThreadLocal` 대체 — Virtual Thread 환경에서 더 안전하고 효율적

```java
public static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

// Filter에서 바인딩
ScopedValue.runWhere(CONTEXT, requestContext, () -> {
    filterChain.doFilter(request, response);
});

// Service에서 읽기
RequestContext ctx = CONTEXT.get();
```

### Stable Values (Java 25)
- 지연 초기화가 필요한 불변 필드에 `StableValue` 사용

```java
public class ProductCacheService {
    private final StableValue<Map<String, Product>> cache = StableValue.of();

    public Map<String, Product> getProductCache() {
        return cache.orElseSet(this::loadAllProducts);
    }
}
```

## 네이밍 규칙

| 대상 | 규칙 | 예시 |
|:---|:---|:---|
| 패키지 | 소문자, 단수형 | `order`, `payment` |
| 클래스 | PascalCase | `CreateOrderService` |
| 메서드 | camelCase, 동사 시작 | `cancel()`, `findById()` |
| 상수 | UPPER_SNAKE_CASE | `MAX_ORDER_ITEMS` |
| 변수 | camelCase | `orderItems` |
| Use Case (명령) | `{Action}{Domain}UseCase` | `CreateOrderUseCase` |
| Use Case (조회) | `{Domain}{Action}Query` | `OrderFindQuery` |
| Service (구현체) | `{Action}{Domain}Service` | `CreateOrderService` |
| Command | `{Action}{Domain}Command` | `CreateOrderCommand` |
| Outbound Port | `{Action}{Domain}Port` | `SaveOrderPort`, `LoadOrderPort` |
| Controller | `{Domain}Controller` | `OrderController` |
| JPA Entity | `{Domain}JpaEntity` | `OrderJpaEntity` |
| Mapper | `{Domain}PersistenceMapper` | `OrderPersistenceMapper` |
| Request DTO | `{Action}{Domain}Request` | `CreateOrderRequest` |
| Response DTO | `{Domain}Response` | `OrderResponse` |
| 공개 이벤트 | `{Domain}{Action}Event` | `OrderCreatedEvent` |
| Exception | `{Domain}{상황}Exception` | `OrderNotFoundException` |
| Fixture | `{Domain}Fixture` | `OrderFixture` |
| 모듈 설정 | `{Domain}ModuleConfig` | `OrderModuleConfig` |
| 이벤트 핸들러 | `{Source}{Event}Handler` | `PaymentCompletedEventHandler` |
| 공개 조회 API | `Find{Domain}Query` | `FindProductQuery` |

## 불변성 원칙

- Domain Model: setter 전면 금지, 상태 변경은 도메인 메서드로
- DTO/Command/Query: record 사용 (기본 불변)
- 컬렉션 반환: `List.copyOf()`, `Collections.unmodifiable*()` 사용
- 날짜/시간: `LocalDateTime`, `Instant` 등 java.time API만 사용
- 이벤트: record + 불변 필드로 모듈 간 안전한 전달 보장

## Optional 사용 규칙

```java
// ✅ 허용: 반환 타입
Optional<Order> findById(OrderId id);

// ❌ 금지: 필드, 메서드 파라미터, 컬렉션 감싸기
```

## Spring Boot 4.x 규칙

- `jakarta.*` 네임스페이스 사용 (`javax.*` 금지)
- HTTP 클라이언트: `RestClient` 사용 (`RestTemplate`, `WebClient` 대신)
- Bean 등록: 각 모듈에 `{Domain}ModuleConfig` 클래스에서 명시적 `@Bean` 등록
- 컴포넌트 스캔: `_shared/` 에서만 최소한으로 사용
- 설정 파일: `application.yml`만 사용 (`.properties` 금지)
- Virtual Thread: `spring.threads.virtual.enabled=true` 활성화 상태

## Spring Modulith 규칙

- 이벤트 수신: `@ApplicationModuleListener` 사용 (자동으로 `@Async` + `@TransactionalEventListener(AFTER_COMMIT)`)
- 모듈 설정: 각 모듈 루트에 `{Domain}ModuleConfig.java` 배치
- 모듈 선언: `package-info.java`에 `@ApplicationModule` 어노테이션
- 공개 API: `api/` 패키지에만 다른 모듈이 사용할 이벤트/인터페이스 배치

## Lombok 사용 제한

- `@RequiredArgsConstructor`: Service, Adapter 등 생성자 주입에만 허용
- `@Getter`: JPA Entity에만 허용 (domain model에는 금지)
- `@Builder`: 테스트 Fixture 클래스에서만 허용
- `@Setter`, `@Data`, `@Value`: 전면 금지
- `@Slf4j`: 허용
- Domain Model은 Lombok 전면 금지 — 순수 Java로 작성
