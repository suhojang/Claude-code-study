---
description: 성능 최적화 및 운영 규칙. 쿼리, 캐시, 비동기 처리 코드 작성 시 적용.
globs: src/main/java/**/*.java
---

# 성능 & 최적화 규칙

## JPA 쿼리 최적화

### N+1 문제 방지
```java
// ✅ Fetch Join으로 해결
@Query("SELECT o FROM OrderJpaEntity o JOIN FETCH o.items WHERE o.id = :id")
Optional<OrderJpaEntity> findByIdWithItems(@Param("id") Long id);

// ✅ @EntityGraph로 해결
@EntityGraph(attributePaths = {"items", "items.product"})
Optional<OrderJpaEntity> findWithItemsById(Long id);

// ❌ 금지: Lazy Loading 후 루프에서 접근 (N+1 발생)
List<OrderJpaEntity> orders = orderRepository.findAll();
orders.forEach(o -> o.getItems().size());  // ❌ N+1
```

### 대량 데이터 조회
- 목록 조회 시 반드시 페이징 적용 (`Pageable`)
- 필요한 컬럼만 조회: Interface Projection 또는 DTO Projection

```java
// ✅ DTO Projection: 필요한 필드만 조회
@Query("""
    SELECT new com.example.app.order.adapter.out.persistence.dto.OrderSummary(
        o.id, o.status, o.totalAmount, o.createdAt
    )
    FROM OrderJpaEntity o
    WHERE o.memberId = :memberId
    """)
Page<OrderSummary> findSummaryByMemberId(@Param("memberId") Long memberId, Pageable pageable);
```

### 벌크 연산
```java
// ✅ Bulk Update: 대량 상태 변경 시 개별 save 금지
@Modifying(clearAutomatically = true)
@Query("UPDATE OrderJpaEntity o SET o.status = :status WHERE o.id IN :ids")
int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);
```

## 캐시 전략 (Redis)

### 캐시 적용 대상
- 변경이 드문 조회 데이터: 상품 상세, 카테고리 목록
- 비용이 큰 집계 쿼리: 인기 상품 순위, 매출 통계

### 캐시 미적용 대상
- 실시간 정합성이 중요한 데이터: 재고 수량, 결제 상태
- 사용자별 개인화 데이터: 장바구니 (별도 Redis 구조 사용)

```java
// ✅ Outbound Port에서 캐시 적용 (domain은 캐시를 모름)
@Repository
@RequiredArgsConstructor
class ProductPersistenceAdapter implements LoadProductPort {

    private final ProductJpaRepository productJpaRepository;
    private final ProductPersistenceMapper mapper;
    private final RedisTemplate<String, ProductCacheDto> redisTemplate;

    @Override
    public Optional<Product> findById(ProductId id) {
        String key = "product:" + id.value();
        ProductCacheDto cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return Optional.of(cached.toDomain());
        }
        return productJpaRepository.findById(id.value())
            .map(entity -> {
                Product product = mapper.toDomain(entity);
                redisTemplate.opsForValue().set(key, ProductCacheDto.from(product),
                    Duration.ofMinutes(30));
                return product;
            });
    }
}
```

### 캐시 규칙
- 캐시 로직은 Adapter 계층에서만 구현 (Domain은 캐시 존재를 모른다)
- TTL 필수 설정 (무기한 캐시 금지)
- 데이터 변경 시 관련 캐시 무효화 (`@CacheEvict` 또는 명시적 삭제)

## Virtual Thread & Structured Concurrency (Java 25 + Spring Boot 4.x)

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

### Virtual Thread 규칙
- I/O 바운드 작업(DB, HTTP, 파일)에서 자동 활용
- `synchronized` 블록 사용 자제 → `ReentrantLock` 사용 (pinning 방지)
- `ThreadLocal` 사용 금지 → `ScopedValue` (Java 25 정식) 사용

```java
// ❌ synchronized → Virtual Thread pinning 발생
synchronized (this) { }

// ✅ ReentrantLock 사용
private final ReentrantLock lock = new ReentrantLock();
public void doSomething() {
    lock.lock();
    try { /* ... */ } finally { lock.unlock(); }
}
```

### ScopedValue (ThreadLocal 대체)
```java
// ❌ ThreadLocal: Virtual Thread에서 메모리 누수 위험
private static final ThreadLocal<MemberId> currentMember = new ThreadLocal<>();

// ✅ ScopedValue: Virtual Thread 안전, 불변, 범위 자동 해제
private static final ScopedValue<MemberId> CURRENT_MEMBER = ScopedValue.newInstance();

ScopedValue.runWhere(CURRENT_MEMBER, memberId, () -> {
    orderService.execute(command);
});
```

### Structured Concurrency (병렬 작업)
```java
// ✅ 외부 API 병렬 호출 — 하나라도 실패하면 나머지 자동 취소
public OrderSummary buildOrderSummary(OrderId orderId) throws Exception {
    try (var scope = StructuredTaskScope.open()) {
        var orderTask = scope.fork(() -> loadOrderPort.findById(orderId));
        var paymentTask = scope.fork(() -> loadPaymentPort.findByOrderId(orderId));
        var deliveryTask = scope.fork(() -> trackDeliveryPort.track(orderId));
        scope.join();
        return new OrderSummary(
            orderTask.get(), paymentTask.get(), deliveryTask.get()
        );
    }
}
```

### StableValue (지연 초기화)
```java
// ✅ StableValue: 스레드 안전한 지연 초기화 (Java 25)
private final StableValue<ExpensiveResource> resource = StableValue.of();

public ExpensiveResource getResource() {
    return resource.orElseSet(this::initializeResource);
}
```

## Spring Modulith 이벤트 성능

### @ApplicationModuleListener 비동기 처리
```java
// Spring Modulith의 @ApplicationModuleListener는 기본적으로
// @Async + @TransactionalEventListener(AFTER_COMMIT) 조합
// Virtual Thread가 활성화되면 자동으로 Virtual Thread에서 실행

@ApplicationModuleListener
void on(OrderCreatedEvent event) {
    // 비동기로 실행 — 발행자의 응답 속도에 영향 없음
    notificationService.sendOrderConfirmation(event.orderId());
}
```

### 이벤트 재시도 (Event Publication Registry)
```yaml
spring:
  modulith:
    events:
      jdbc:
        schema-initialization:
          enabled: true
    republish-outstanding-events-on-restart: true
```

## 외부 API 호출 (Resilience)

```java
@Repository
@RequiredArgsConstructor
class PaymentGatewayAdapter implements RequestPaymentPort {

    private final RestClient restClient;

    @Override
    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
    @TimeLimiter(name = "paymentGateway")
    @Retry(name = "paymentGateway")
    public PaymentResult request(PaymentRequest request) {
        return restClient.post()
            .uri("/api/payments")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }

    private PaymentResult fallback(PaymentRequest request, Throwable t) {
        throw new ExternalServiceException(PaymentErrorCode.PG_SERVICE_UNAVAILABLE,
            "결제 서비스가 일시적으로 불가합니다");
    }
}
```

### Resilience 규칙
- 외부 API 호출에는 반드시 Circuit Breaker 적용
- Timeout: 기본 3초, 최대 5초
- Retry: 최대 3회, 지수 백오프 (1s, 2s, 4s)
- Fallback: 적절한 DomainException throw 또는 기본값 반환

## 로깅

```java
// ✅ 구조화된 로그
log.info("주문 생성 완료 [orderId={}, memberId={}, amount={}]",
    orderId, memberId, totalAmount);

// ❌ 금지: 민감 정보 로깅, 문자열 연결
```

### 로깅 규칙
- SLF4J 플레이스홀더 `{}` 사용 (문자열 연결 금지)
- 민감 정보(비밀번호, 토큰, 카드번호) 로그 출력 금지
- 비즈니스 이벤트: `info`, 경고: `warn`, 예외: `error` (스택트레이스 포함)
