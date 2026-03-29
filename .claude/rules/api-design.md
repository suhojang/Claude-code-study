---
description: REST API 설계 및 Web Adapter 규칙. Controller와 DTO 작성 시 적용.
globs: src/main/java/**/adapter/in/web/**/*.java
---

# REST API 설계 규칙

## URL 설계

- 기본 경로: `/api/v1/{도메인복수형}` (예: `/api/v1/orders`, `/api/v1/products`)
- API 버전: URL 경로 방식 (`/api/v1/`, `/api/v2/`)
- 리소스명: 복수형 명사, kebab-case (`/api/v1/order-items`)
- 행위(동사)는 HTTP Method로 표현, URL에 동사 금지

```
✅ POST   /api/v1/orders              → 주문 생성
✅ GET    /api/v1/orders/{orderId}    → 주문 단건 조회
✅ GET    /api/v1/orders              → 주문 목록 조회 (페이징)
✅ PATCH  /api/v1/orders/{orderId}    → 주문 부분 수정
✅ DELETE /api/v1/orders/{orderId}    → 주문 삭제

❌ POST   /api/v1/orders/create       → URL에 동사 금지
❌ GET    /api/v1/getOrders           → URL에 동사 금지
```

### 비 CRUD 행위가 필요한 경우

```
POST /api/v1/orders/{orderId}/cancel       → 주문 취소
POST /api/v1/orders/{orderId}/confirm      → 주문 확정
POST /api/v1/payments/{paymentId}/refund   → 결제 환불
```

## HTTP 상태 코드

| 상태 코드 | 사용 상황 |
|:---|:---|
| `200 OK` | 조회 성공, 수정 성공 |
| `201 Created` | 리소스 생성 성공 (Location 헤더에 생성된 리소스 URI 포함) |
| `204 No Content` | 삭제 성공, 응답 본문 없음 |
| `400 Bad Request` | Bean Validation 실패, 잘못된 요청 형식 |
| `401 Unauthorized` | 인증 실패 (토큰 없음/만료) |
| `403 Forbidden` | 인가 실패 (권한 없음) |
| `404 Not Found` | 리소스 미존재 |
| `409 Conflict` | 비즈니스 규칙 충돌 (중복 주문, 재고 부족 등) |
| `500 Internal Server Error` | 서버 내부 오류 (의도하지 않은 예외) |

## 공통 응답 래퍼

### 성공 응답

```java
public record ApiResponse<T>(
    int code,
    String message,
    T data
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(200, "OK", data);
    }

    public static <T> ApiResponse<T> created(T data) {
        return new ApiResponse<>(201, "Created", data);
    }

    public static ApiResponse<Void> noContent() {
        return new ApiResponse<>(204, "No Content", null);
    }
}
```

### 페이징 응답

```java
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean hasNext
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.hasNext()
        );
    }
}
```

## Controller 패턴

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;
    private final OrderFindQuery orderFindQuery;
    private final CancelOrderUseCase cancelOrderUseCase;

    @PostMapping
    ResponseEntity<ApiResponse<CreateOrderResponse>> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        OrderId orderId = createOrderUseCase.execute(request.toCommand());

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(ApiResponse.created(new CreateOrderResponse(orderId.value())));
    }

    @GetMapping("/{orderId}")
    ApiResponse<OrderResponse> findOrder(@PathVariable Long orderId) {
        Order order = orderFindQuery.findById(OrderId.of(orderId));
        return ApiResponse.ok(OrderResponse.from(order));
    }

    @GetMapping
    ApiResponse<PageResponse<OrderResponse>> findOrders(
            @AuthenticationPrincipal MemberPrincipal principal,
            Pageable pageable) {

        Page<Order> orders = orderFindQuery.findByMemberId(
            MemberId.of(principal.id()), pageable);
        Page<OrderResponse> responses = orders.map(OrderResponse::from);

        return ApiResponse.ok(PageResponse.from(responses));
    }

    @PostMapping("/{orderId}/cancel")
    ApiResponse<Void> cancelOrder(
            @PathVariable Long orderId,
            @Valid @RequestBody CancelOrderRequest request) {

        cancelOrderUseCase.execute(request.toCommand(OrderId.of(orderId)));
        return ApiResponse.ok(null);
    }
}
```

## Request / Response DTO

### Request DTO

```java
public record CreateOrderRequest(
    @NotNull(message = "회원 ID는 필수입니다")
    Long memberId,

    @NotEmpty(message = "주문 항목은 최소 1개 이상이어야 합니다")
    @Valid
    List<OrderItemRequest> items,

    @NotNull @Valid
    ShippingAddressRequest shippingAddress
) {
    // Request → Command 변환은 DTO 내부에서
    public CreateOrderCommand toCommand() {
        return new CreateOrderCommand(
            MemberId.of(memberId),
            items.stream().map(OrderItemRequest::toCommand).toList(),
            shippingAddress.toDomain()
        );
    }
}
```

### Response DTO

```java
public record OrderResponse(
    Long orderId,
    String status,
    Long totalAmount,
    List<OrderItemResponse> items,
    LocalDateTime createdAt
) {
    // Domain Model → Response 변환은 정적 팩토리 메서드
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId().value(),
            order.getStatus().name(),
            order.getTotalAmount().value(),
            order.getItems().stream().map(OrderItemResponse::from).toList(),
            order.getCreatedAt()
        );
    }
}
```

## 규칙 요약

1. Controller에 비즈니스 로직 금지 — DTO 변환과 UseCase 호출만 수행
2. Domain Model을 응답에 직접 반환 금지 — 반드시 Response DTO로 변환
3. Bean Validation은 Request DTO에만 적용
4. `@Valid`로 중첩 객체까지 검증
5. 인증 정보는 `@AuthenticationPrincipal`로 주입
6. Controller 메서드의 반환 타입은 `ApiResponse<T>` 또는 `ResponseEntity<ApiResponse<T>>`
