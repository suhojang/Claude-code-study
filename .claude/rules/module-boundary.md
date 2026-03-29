---
description: Spring Modulith 기반 모듈 경계 및 팀 간 협업 충돌 방지 규칙. 모든 Java 소스 코드에 적용.
globs: src/main/java/**/*.java
---

# 모듈 경계 규칙 (Spring Modulith + 협업 충돌 방지)

## 핵심 원칙: 모듈 = 팀의 소유 단위

각 Application Module은 하나의 팀(또는 담당자)이 소유한다.
**다른 팀의 모듈 내부(`domain/`, `adapter/`)를 직접 수정하거나 참조하면 안 된다.**
통신은 반드시 공개 API(`api/` 패키지)와 이벤트를 통해서만 이루어진다.

## 모듈 간 참조 허용/금지 매트릭스

| 참조 원본 → 대상 | `api/` 패키지 | `domain/` | `adapter/` | 모듈 루트 |
|:---|:---:|:---:|:---:|:---:|
| **같은 모듈** | ✅ | ✅ | ✅ | ✅ |
| **다른 모듈** | ✅ 이벤트/인터페이스만 | ❌ 접근 불가 | ❌ 접근 불가 | ✅ Config만 |
| **`_shared`** | ✅ | ✅ | ✅ | ✅ |

## Spring Modulith 가시성 강제

Spring Modulith는 모듈 내부 패키지(`domain/`, `adapter/`)를 자동으로 **internal**로 취급한다.
다른 모듈에서 내부 패키지를 import하면 `ModularityTests`에서 즉시 실패한다.

```java
// ✅ 허용: 다른 모듈의 공개 이벤트 참조
import com.example.app.order.api.OrderCreatedEvent;

// ✅ 허용: 다른 모듈의 공개 조회 인터페이스 참조
import com.example.app.product.api.FindProductQuery;

// ❌ 금지 (Spring Modulith가 자동 차단): 다른 모듈의 내부 참조
import com.example.app.order.domain.model.Order;           // 내부 모델
import com.example.app.order.domain.service.OrderService;   // 내부 서비스
import com.example.app.order.adapter.out.persistence.*;     // 내부 어댑터
```

## 모듈 간 통신 패턴

### 패턴 1: 이벤트 기반 통신 (기본, 권장)

모듈 간 느슨한 결합의 핵심. 대부분의 모듈 간 통신에 사용한다.

```java
// === order 모듈: 이벤트 발행 ===

// order/api/OrderCreatedEvent.java (공개 API)
public record OrderCreatedEvent(
    Long orderId,
    Long memberId,
    Long totalAmount,
    Instant occurredAt
) {}

// order/adapter/out/event/OrderEventPublisherAdapter.java (내부)
@Component
@RequiredArgsConstructor
class OrderEventPublisherAdapter implements OrderEventPort {
    private final ApplicationEventPublisher publisher;

    @Override
    public void publishOrderCreated(Order order) {
        publisher.publishEvent(new OrderCreatedEvent(
            order.getId().value(), order.getMemberId().value(),
            order.getTotalAmount().value(), Instant.now()
        ));
    }
}
```

```java
// === payment 모듈: 이벤트 수신 ===

// payment/adapter/in/event/OrderCreatedEventHandler.java (내부)
@Component
@RequiredArgsConstructor
class OrderCreatedEventHandler {

    private final CreatePaymentUseCase createPaymentUseCase;

    @ApplicationModuleListener   // Spring Modulith: 트랜잭션 커밋 후 비동기 실행
    void on(OrderCreatedEvent event) {
        createPaymentUseCase.execute(
            new CreatePaymentCommand(event.orderId(), event.totalAmount())
        );
    }
}
```

### 패턴 2: 공개 인터페이스 (동기 조회가 반드시 필요한 경우)

동기 조회가 필수인 경우에만 제한적으로 사용한다.

```java
// product/api/FindProductQuery.java (공개 API)
public interface FindProductQuery {
    ProductInfo findById(Long productId);
    List<ProductInfo> findByIds(List<Long> productIds);

    record ProductInfo(
        Long id,
        String name,
        long price,
        int stockQuantity
    ) {}
}

// product/domain/service/FindProductService.java (내부 구현)
@UseCase
@RequiredArgsConstructor
class FindProductService implements FindProductQuery {

    private final LoadProductPort loadProductPort;

    @Override
    public ProductInfo findById(Long productId) {
        Product product = loadProductPort.findById(ProductId.of(productId))
            .orElseThrow(() -> new ProductNotFoundException(ProductId.of(productId)));
        return new ProductInfo(
            product.getId().value(), product.getName(),
            product.getPrice().value(), product.getStockQuantity()
        );
    }
}
```

```java
// order 모듈에서 사용
@UseCase
@RequiredArgsConstructor
class CreateOrderService implements CreateOrderUseCase {

    private final FindProductQuery findProductQuery;  // ✅ 공개 인터페이스 주입

    @Override
    public OrderId execute(CreateOrderCommand command) {
        var product = findProductQuery.findById(command.productId());
        // ...
    }
}
```

### 패턴 선택 기준

| 상황 | 패턴 | 이유 |
|:---|:---|:---|
| 주문 생성 → 결제 요청 | 이벤트 | 비동기, 실패 시 재시도 가능 |
| 주문 생성 → 재고 차감 | 이벤트 | 비동기, 보상 트랜잭션 가능 |
| 주문 생성 시 상품 정보 조회 | 공개 인터페이스 | 동기 조회 필수, 즉시 결과 필요 |
| 주문 완료 → 알림 발송 | 이벤트 | 비동기, fire-and-forget |

## Spring Modulith 이벤트 고급 설정

### 이벤트 발행 + 완료 보장 (Event Publication Registry)

```java
// Spring Modulith는 이벤트를 DB에 기록하여 전달 보장
// application.yml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true
    republish-outstanding-events-on-restart: true
```

### 이벤트 외부화 (Kafka 연동 준비)

```java
// 마이크로서비스 전환 시 이벤트를 Kafka로 외부화
@Configuration
class EventExternalizationConfig {

    @Bean
    EventExternalizationConfiguration orderEvents() {
        return EventExternalizationConfiguration.externalizing()
            .select(EventExternalizationConfiguration.annotatedAsExternalized())
            .build();
    }
}
```

## 팀 협업 충돌 방지 전략

### 1. 파일 소유권 분리 (CODEOWNERS)

```
# .github/CODEOWNERS
/src/main/java/com/example/app/order/       @team-order
/src/main/java/com/example/app/product/     @team-product
/src/main/java/com/example/app/member/      @team-member
/src/main/java/com/example/app/payment/     @team-payment
/src/main/java/com/example/app/delivery/    @team-delivery
/src/main/java/com/example/app/_shared/     @team-platform

/src/main/resources/db/migration/order/      @team-order
/src/main/resources/db/migration/product/    @team-product
```

### 2. 공유 파일 최소화

| 파일 | 충돌 위험 | 대응 전략 |
|:---|:---:|:---|
| `application.yml` | 높음 | 모듈별 `application-{module}.yml` 분리 + `@PropertySource` |
| `SecurityConfig` | 중간 | URL 패턴을 모듈별 Config에서 `SecurityCustomizer` Bean으로 기여 |
| `build.gradle.kts` | 중간 | 모듈별 의존성을 별도 .gradle 파일로 분리 (`apply from`) |
| `_shared/` | 낮음 | 변경 빈도 낮게 유지, platform 팀만 수정 |

### 3. 모듈별 설정 분리

```java
// order/OrderModuleConfig.java — 주문 모듈 전용 설정
@Configuration
class OrderModuleConfig {

    @Bean
    SecurityCustomizer orderSecurityCustomizer() {
        return http -> http
            .requestMatchers("/api/v1/orders/**").hasRole("USER");
    }
}
```

```java
// global SecurityConfig에서 모듈별 Customizer를 수집
@Configuration
@RequiredArgsConstructor
class SecurityConfig {

    private final List<SecurityCustomizer> customizers;

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // 각 모듈이 기여한 보안 규칙을 자동 적용
        for (SecurityCustomizer customizer : customizers) {
            customizer.customize(http);
        }
        return http.build();
    }
}
```

### 4. 충돌 발생 시나리오별 해결

| 시나리오 | 원인 | 해결 |
|:---|:---|:---|
| 두 팀이 같은 yml 수정 | 공유 설정 파일 | 모듈별 yml 분리 |
| 두 팀이 같은 테이블 참조 | DB 스키마 공유 | 각 모듈은 자기 접두어 테이블만 소유, 조회는 이벤트/API |
| 새 모듈이 기존 모듈 의존 | 동기 호출 필요 | `api/` 공개 인터페이스 또는 이벤트 사용 |
| 공통 DTO 변경 | `_shared/` 수정 | `_shared/` 변경은 platform 팀만, 하위 호환 필수 |

## ArchUnit + Spring Modulith 검증

```java
@AnalyzeClasses(packages = "com.example.app")
class ModuleBoundaryArchTest {

    // Spring Modulith 모듈 구조 자동 검증
    ApplicationModules modules = ApplicationModules.of(Application.class);

    @Test
    void should_verify_modulith_structure() {
        modules.verify();  // 내부 패키지 접근 위반, 순환 참조 등 자동 감지
    }

    // 추가 ArchUnit 규칙: domain → adapter 역방향 의존 금지
    @ArchTest
    static final ArchRule domain_should_not_depend_on_adapter =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..adapter..");

    // 추가 ArchUnit 규칙: domain에 프레임워크 어노테이션 금지
    @ArchTest
    static final ArchRule domain_model_should_be_framework_free =
        noClasses().that().resideInAPackage("..domain.model..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "jakarta.persistence..",
                "org.springframework..",
                "com.fasterxml.jackson.."
            );
}
```

## 모듈 문서 자동 생성

```java
// 모듈 간 의존 관계를 문서로 자동 생성
@Test
void should_generate_module_documentation() {
    ApplicationModules modules = ApplicationModules.of(Application.class);
    new Documenter(modules)
        .writeModulesAsPlantUml()           // PlantUML 다이어그램
        .writeIndividualModulesAsPlantUml() // 모듈별 상세 다이어그램
        .writeModuleCanvases();             // 모듈 캔버스 (입출력 정리)
}
```
