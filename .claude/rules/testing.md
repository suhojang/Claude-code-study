---
description: 테스트 코드 작성 규칙. 모든 테스트 파일에 적용.
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
| 모듈 간 통합 | 통합 테스트 | `@SpringBootTest` + Testcontainers | 전체 로드 |
| 아키텍처 검증 | ArchUnit | `@AnalyzeClasses` | ❌ |

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

@DisplayName("이미 취소된 주문을 다시 취소하면 예외가 발생한다")
@Test
void should_throw_exception_when_cancel_already_cancelled_order() {
    // Given
    // When & Then
}
```

## Given-When-Then 패턴

모든 테스트에 주석으로 구분. 각 섹션의 역할을 명확히 한다.

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

    // Mock은 Mockito로 직접 생성
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
@SpringBootTest  // ❌ domain 테스트에 사용 금지
class CreateOrderServiceTest { }

@ExtendWith(SpringExtension.class)  // ❌ 불필요한 Spring 확장
class CreateOrderServiceTest { }
```

## Web Adapter 테스트

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private CreateOrderUseCase createOrderUseCase;  // Port 인터페이스를 Mock

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
        // 리플렉션 또는 테스트용 메서드로 상태 변경
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

- 하나의 테스트에 하나의 검증 (단일 assert 또는 밀접하게 관련된 assert 그룹)
- 테스트 간 순서 의존성 금지 (독립 실행 가능해야 함)
- 테스트 데이터는 테스트 내부에서 생성 (외부 파일/DB 의존 최소화)
- `@Nested` 클래스로 테스트를 논리적으로 그룹핑

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
