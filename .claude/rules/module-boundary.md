---
description: 모놀리스 내 도메인 모듈 간 경계 규칙. 모듈 간 참조 코드 작성 시 적용.
globs: src/main/java/**/*.java
---

# 모듈 경계 규칙 (Module Boundary)

모놀리스에서 헥사고날 아키텍처의 가치를 지키는 가장 중요한 규칙.
이 규칙이 무너지면 모듈 간 강결합으로 인해 마이크로서비스 전환이 불가능해진다.

## 모듈 간 참조 허용/금지 매트릭스

| 참조 원본 → 대상 | domain.port.in | domain.model | domain.service | adapter |
|:---|:---:|:---:|:---:|:---:|
| **같은 모듈** | ✅ | ✅ | ✅ | ✅ |
| **다른 모듈의 domain** | ✅ Port.in만 | ❌ | ❌ | ❌ |
| **다른 모듈의 adapter** | ❌ | ❌ | ❌ | ❌ |
| **global** | ✅ | ✅ | ✅ | ✅ |

## 핵심 원칙

### 1. 다른 모듈은 Inbound Port(UseCase/Query)로만 접근
```java
// ✅ 올바른 모듈 간 참조
// order 모듈의 서비스에서 product 모듈 조회
@UseCase
@RequiredArgsConstructor
public class CreateOrderService implements CreateOrderUseCase {

    private final FindProductPort findProductPort;    // ✅ product의 Outbound Port? ❌
    private final ProductFindQuery productFindQuery;  // ✅ product의 Inbound Port (Query)

    @Override
    public OrderId execute(CreateOrderCommand command) {
        ProductInfo product = productFindQuery.findById(command.productId());
        // ...
    }
}
```

```java
// ❌ 금지: 다른 모듈의 구현체 직접 참조
import com.example.app.product.domain.service.ProductService;         // ❌
import com.example.app.product.domain.model.Product;                  // ❌
import com.example.app.product.adapter.out.persistence.ProductJpaRepository; // ❌
```

### 2. 다른 모듈의 Domain Model 직접 사용 금지
```java
// ✅ 올바른 방식: Query가 전용 응답 DTO(record)를 반환
public interface ProductFindQuery {
    ProductInfo findById(ProductId id);

    // 다른 모듈에 노출할 최소한의 정보만 담은 record
    record ProductInfo(
        ProductId id,
        String name,
        Money price,
        int stockQuantity
    ) {}
}
```

```java
// ❌ 금지: 다른 모듈의 도메인 모델을 직접 반환
public interface ProductFindQuery {
    Product findById(ProductId id);  // ❌ Product는 product 모듈의 내부 모델
}
```

### 3. 순환 참조는 ApplicationEvent로 해소

모듈 A → 모듈 B, 모듈 B → 모듈 A 양방향 참조가 필요한 경우:

```java
// order 모듈: 주문 완료 이벤트 발행
public record OrderCompletedEvent(
    OrderId orderId,
    MemberId memberId,
    Money totalAmount,
    LocalDateTime completedAt
) {}

// payment 모듈: 이벤트 수신하여 처리
@Component
@RequiredArgsConstructor
public class OrderCompletedEventListener {

    private final ProcessPaymentUseCase processPaymentUseCase;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCompletedEvent event) {
        processPaymentUseCase.execute(
            new ProcessPaymentCommand(event.orderId(), event.totalAmount())
        );
    }
}
```

### 4. global 패키지의 독립성

- `global/`은 어떤 도메인 모듈도 import하지 않는다
- 도메인 모듈은 `global/`의 공통 코드를 자유롭게 사용 가능
- `global/`에 비즈니스 로직이 들어가면 안 된다

### 5. 공유 식별자 (Shared Kernel)

모듈 간 공유가 필요한 ID 타입은 각 모듈이 자체 정의한다:

```java
// order 모듈 내부
public record MemberId(Long value) {}   // order가 사용하는 MemberId

// member 모듈 내부
public record MemberId(Long value) {}   // member가 정의하는 MemberId
```

또는, 정말 범용적인 ID만 `global/common/`에 배치:

```java
// global/common/id/
public record MemberId(Long value) {}   // 전역 공유 ID (최소한으로 유지)
```

## ArchUnit 검증

이 규칙은 ArchUnit 테스트로 자동 검증한다:

```java
@AnalyzeClasses(packages = "com.example.app")
class ModuleBoundaryArchTest {

    @ArchTest
    static final ArchRule order_domain_should_not_depend_on_other_adapters =
        noClasses().that().resideInAPackage("..order.domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "..product.adapter..",
                "..member.adapter..",
                "..payment.adapter..",
                "..delivery.adapter.."
            );

    @ArchTest
    static final ArchRule order_domain_should_not_access_other_domain_models =
        noClasses().that().resideInAPackage("..order.domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "..product.domain.model..",
                "..product.domain.service..",
                "..member.domain.model..",
                "..member.domain.service.."
            );
}
```
