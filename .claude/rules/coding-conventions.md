---
description: Java 21 + Spring Boot 4.x 코딩 컨벤션. 모든 Java 소스 코드에 적용.
globs: src/main/java/**/*.java
---

# 코딩 컨벤션

## Java 21 기능 활용

### Record
- DTO, Command, Query, Value Object, Event에 적극 사용
- record의 Compact Constructor로 입력값 검증 수행

```java
public record CreateOrderCommand(
    MemberId memberId,
    List<OrderItemCommand> items,
    ShippingAddress shippingAddress
) {
    // Compact Constructor: 검증 로직
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

```java
return switch (event) {
    case OrderCreated e -> handleCreated(e);
    case OrderCancelled e -> handleCancelled(e);
    case OrderCompleted e -> handleCompleted(e);
};
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
| Event | `{Domain}{Action}Event` | `OrderCreatedEvent` |
| Exception | `{Domain}{상황}Exception` | `OrderNotFoundException` |
| Fixture | `{Domain}Fixture` | `OrderFixture` |

## 불변성 원칙

- Domain Model: setter 전면 금지, 상태 변경은 도메인 메서드로
- DTO/Command/Query: record 사용 (기본 불변)
- 컬렉션 반환: `List.copyOf()`, `Collections.unmodifiable*()` 사용
- 날짜/시간: `LocalDateTime`, `Instant` 등 java.time API만 사용

## Optional 사용 규칙

```java
// ✅ 허용: 반환 타입
Optional<Order> findById(OrderId id);

// ❌ 금지: 필드
private Optional<String> nickname;

// ❌ 금지: 메서드 파라미터
void update(Optional<String> name);

// ❌ 금지: 컬렉션 감싸기
Optional<List<Order>> findAll();
```

## Spring Boot 4.x 규칙

- `jakarta.*` 네임스페이스 사용 (`javax.*` 금지)
- HTTP 클라이언트: `RestClient` 사용 (`RestTemplate`, `WebClient` 대신)
- Bean 등록: 각 모듈에 `{Domain}Config` 클래스에서 명시적 `@Bean` 등록
- 컴포넌트 스캔: `global/` 에서만 최소한으로 사용
- 설정 파일: `application.yml`만 사용 (`.properties` 금지)
- Virtual Thread: `spring.threads.virtual.enabled=true` 활성화 상태

## Lombok 사용 제한

- `@RequiredArgsConstructor`: Service, Adapter 등 생성자 주입에만 허용
- `@Getter`: JPA Entity에만 허용 (domain model에는 금지)
- `@Builder`: 테스트 Fixture 클래스에서만 허용
- `@Setter`, `@Data`, `@Value`: 전면 금지
- `@Slf4j`: 허용
- Domain Model은 Lombok 전면 금지 — 순수 Java로 작성
