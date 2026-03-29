---
description: 테스트 코드 작성 규칙. Spring Modulith 모듈 테스트 포함. 모든 테스트 파일에 적용.
globs: src/test/**/*.java
---

# 테스트 전략

## 계층별 테스트 방침

| 계층 | 테스트 종류 | 어노테이션 | Spring Context |
|:---|:---|:---|:---:|
| domain.service | 순수 단위 테스트 | 없음 (Plain JUnit) | ❌ 금지 |
| domain.model | 순수 단위 테스트 | 없음 | ❌ 금지 |
| adapter.in.web | 슬라이스 테스트 | `@WebMvcTest` | 최소 로드 |
| adapter.out.persistence | 슬라이스 테스트 | `@DataJpaTest` + Testcontainers | 최소 로드 |
| 모듈 내부 통합 | 모듈 격리 테스트 | `@ApplicationModuleTest` | 모듈만 로드 |
| 모듈 간 이벤트 | 이벤트 발행/수신 테스트 | `@ApplicationModuleTest` + `Scenario` | 모듈만 로드 |
| 전체 통합 | 통합 테스트 | `@SpringBootTest` + Testcontainers | 전체 로드 |
| 모듈 구조 검증 | Modulith 구조 테스트 | `ApplicationModules.of()` | ❌ |
| 아키텍처 검증 | ArchUnit | `@AnalyzeClasses` | ❌ |

## Spring Modulith 모듈 구조 검증 테스트

모든 모듈 경계 규칙을 자동으로 검증하는 필수 테스트.
**이 테스트가 실패하면 빌드가 중단된다.**

```java
class ModularityTests {

    @Test
    void should_verify_modulith_structure() {
        // 모듈 간 내부 패키지 접근 위반, 순환 참조 등 자동 감지
        ApplicationModules.of(Application.class).verify();
    }

    @Test
    void should_generate_module_documentation() {
        var modules = ApplicationModules.of(Application.class);
        new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml();
    }
}
```

## Spring Modulith 모듈 격리 테스트 (`@ApplicationModuleTest`)

각 모듈을 독립적으로 테스트하여 **다른 모듈 변경에 영향받지 않음**을 보장한다.
팀 간 협업 시 자기 모듈의 테스트만 실행해도 안전성을 확인할 수 있다.

```java
// order 모듈만 격리하여 테스트 (다른 모듈 Bean은 로드하지 않음)
@ApplicationModuleTest
class OrderModuleIntegrationTest {

    @Autowired
    private CreateOrderUseCase createOrderUseCase;

    @MockitoBean
    private FindProductQuery findProductQuery;  // 다른 모듈의 공개 API는 Mock

    @DisplayName("주문 모듈 내부에서 주문 생성이 정상 동작한다")
    @Test
    void should_create_order_within_module() {
        // Given
        given(findProductQuery.findById(anyLong()))
            .willReturn(new ProductInfo(1L, "상품", 10000L, 100));

        // When
        OrderId result = createOrderUseCase.execute(OrderFixture.createCommand());

        // Then
        assertThat(result).isNotNull();
    }
}
```

## Spring Modulith 이벤트 시나리오 테스트 (`Scenario`)

모듈 간 이벤트 발행/수신 흐름을 검증한다.

```java
@ApplicationModuleTest
@RequiredArgsConstructor
class OrderEventPublicationTest {

    private final CreateOrderUseCase createOrderUseCase;

    @DisplayName("주문 생성 시 OrderCreatedEvent가 발행된다")
    @Test
    void should_publish_order_created_event(Scenario scenario) {
        scenario.stimulate(() -> createOrderUseCase.execute(OrderFixture.createCommand()))
            .andWaitForEventOfType(OrderCreatedEvent.class)
            .matching(event -> event.orderId() != null)
            .toArriveAndVerify(event -> {
                assertThat(event.totalAmount()).isPositive();
            });
    }
}
```

## 테스트 메서드 네이밍

- `@DisplayName`: 한글로 비즈니스 의도를 명확히 표현
- 메서드명: `should_{결과}_when_{조건}` 영문 패턴

```java
@DisplayName("재고가 충분하면 주문이 성공적으로 생성된다")
@Test
void should_create_order_when_stock_is_sufficient() {
    // Given
    // When
    // Then
}
```

## Given-When-Then 패턴

모든 테스트에 주석으로 구분.

```java
@Test
void should_create_order_when_stock_is_sufficient() {
    // Given: 충분한 재고를 가진 상품과 회원 정보 준비
    var command = OrderFixture.createCommand();
    given(loadProductPort.findById(any())).willReturn(ProductFixture.withStock(10));
    given(saveOrderPort.save(any())).willReturn(OrderId.of(1L));

    // When: 주문 생성 실행
    OrderId result = createOrderService.execute(command);

    // Then: 주문이 저장되고 이벤트가 발행된다
    assertThat(result).isNotNull();
    then(saveOrderPort).should().save(any(Order.class));
    then(orderEventPort).should().publish(any(OrderCreatedEvent.class));
}
```

## Domain 계층 테스트 (순수 단위 테스트)

```java
// ✅ Spring Context 없이 순수 단위 테스트
class CreateOrderServiceTest {

    private final LoadProductPort loadProductPort = mock(LoadProductPort.class);
    private final SaveOrderPort saveOrderPort = mock(SaveOrderPort.class);
    private final OrderEventPort orderEventPort = mock(OrderEventPort.class);

    private final CreateOrderService sut = new CreateOrderService(
        loadProductPort, saveOrderPort, orderEventPort
    );

    @DisplayName("유효한 주문 요청이면 주문을 생성하고 저장한다")
    @Test
    void should_create_and_save_order_when_valid_request() {
        // Given
        // When
        // Then
    }
}
```

```java
// ❌ 금지: domain 테스트에 Spring Context 로드
@SpringBootTest  // ❌
class CreateOrderServiceTest { }
```

## Web Adapter 테스트

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CreateOrderUseCase createOrderUseCase;

    @DisplayName("POST /api/v1/orders - 유효한 요청이면 201 Created를 반환한다")
    @Test
    void should_return_201_when_valid_create_order_request() throws Exception {
        // Given
        given(createOrderUseCase.execute(any())).willReturn(OrderId.of(1L));

        // When & Then
        mockMvc.perform(post("/api/v1/orders")
                .contentType(APPLICATION_JSON)
                .content("""
                    {
                        "memberId": 1,
                        "items": [{"productId": 100, "quantity": 2}]
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.orderId").value(1));
    }
}
```

## Persistence Adapter 테스트

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = NONE)
class OrderPersistenceAdapterTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderJpaRepository orderJpaRepository;

    @Autowired
    private OrderPersistenceMapper mapper;

    private OrderPersistenceAdapter sut;

    @BeforeEach
    void setUp() {
        sut = new OrderPersistenceAdapter(orderJpaRepository, mapper);
    }

    @DisplayName("주문을 저장하면 ID가 채번되어 반환된다")
    @Test
    void should_return_generated_id_when_save_order() {
        // Given
        Order order = OrderFixture.create();

        // When
        OrderId savedId = sut.save(order);

        // Then
        assertThat(savedId).isNotNull();
        assertThat(orderJpaRepository.findById(savedId.value())).isPresent();
    }
}
```

## Fixture 규칙

- 위치: 각 모듈의 `src/test/java/.../fixture/{Domain}Fixture.java`
- 기본값이 채워진 팩토리 메서드 제공
- 특정 상태를 만드는 Named Constructor 패턴 사용

```java
public class OrderFixture {

    public static Order create() {
        return Order.create(
            MemberId.of(1L),
            List.of(OrderItemFixture.create()),
            ShippingAddressFixture.create()
        );
    }

    public static Order withStatus(OrderStatus status) {
        Order order = create();
        ReflectionTestUtils.setField(order, "status", status);
        return order;
    }

    public static CreateOrderCommand createCommand() {
        return new CreateOrderCommand(
            MemberId.of(1L),
            List.of(new OrderItemCommand(ProductId.of(100L), 2)),
            ShippingAddressFixture.create()
        );
    }
}
```

## 테스트 원칙

- 하나의 테스트에 하나의 검증
- 테스트 간 순서 의존성 금지
- 테스트 데이터는 테스트 내부에서 생성
- `@Nested` 클래스로 테스트를 논리적으로 그룹핑
- **모듈 테스트 독립성**: 자기 모듈 테스트는 다른 모듈 변경에 영향받지 않아야 함
- **다른 모듈 의존은 반드시 Mock**: `@ApplicationModuleTest`에서 외부 모듈 공개 API는 `@MockitoBean`

```java
class OrderTest {

    @Nested
    @DisplayName("주문 생성")
    class Create {
        @Test void should_create_order_when_valid_input() { }
        @Test void should_throw_when_empty_items() { }
    }

    @Nested
    @DisplayName("주문 취소")
    class Cancel {
        @Test void should_cancel_order_when_cancellable_status() { }
        @Test void should_throw_when_already_cancelled() { }
    }
}
```
