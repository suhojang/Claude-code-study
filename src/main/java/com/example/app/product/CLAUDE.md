# 상품 모듈 (Product Module)

**소유팀**: team-product
**테이블 접두어**: `prd_`
**API 경로**: `/api/v1/products`

## 모듈 구조

```
product/
├── ProductModuleConfig.java
├── api/                            # ✅ 공개 API
│   ├── FindProductQuery.java       #   상품 조회 공개 인터페이스 (동기)
│   ├── ProductStockDecreasedEvent.java
│   └── ProductStockRestoredEvent.java
├── domain/                         # 🔒 내부 전용
│   ├── model/
│   │   ├── Product.java
│   │   ├── ProductId.java
│   │   ├── Category.java
│   │   ├── ProductStatus.java
│   │   └── StockQuantity.java      # VO: 재고 수량 (음수 검증)
│   ├── port/in/
│   │   ├── RegisterProductUseCase.java
│   │   ├── UpdateProductUseCase.java
│   │   ├── DecreaseStockUseCase.java
│   │   └── ProductFindQuery.java   # 내부 조회
│   ├── port/out/
│   │   ├── SaveProductPort.java
│   │   ├── LoadProductPort.java
│   │   └── ProductEventPort.java
│   ├── service/
│   │   ├── RegisterProductService.java
│   │   ├── UpdateProductService.java
│   │   ├── DecreaseStockService.java
│   │   └── FindProductService.java  # FindProductQuery(공개) + ProductFindQuery(내부) 구현
│   └── exception/
│       ├── ProductErrorCode.java    # PRD-001 ~ PRD-xxx
│       ├── ProductNotFoundException.java
│       └── InsufficientStockException.java
└── adapter/
    ├── in/
    │   ├── web/
    │   │   ├── ProductController.java
    │   │   ├── RegisterProductRequest.java
    │   │   └── ProductResponse.java
    │   └── event/
    │       └── OrderCreatedEventHandler.java    # 주문 생성 시 재고 차감
    └── out/
        ├── persistence/
        │   ├── ProductPersistenceAdapter.java
        │   ├── ProductJpaEntity.java
        │   ├── CategoryJpaEntity.java
        │   ├── ProductJpaRepository.java
        │   └── ProductPersistenceMapper.java
        └── event/
            └── ProductEventPublisherAdapter.java
```

## 공개 API: FindProductQuery

다른 모듈(order 등)이 상품 정보를 **동기적으로 조회**할 때 사용하는 공개 인터페이스.

```java
// product/api/FindProductQuery.java
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
```

## 발행 이벤트

| 이벤트 | 발행 시점 | 소비 모듈 |
|:---|:---|:---|
| `ProductStockDecreasedEvent` | 재고 차감 완료 | notification (품절 임박 알림) |
| `ProductStockRestoredEvent` | 재고 복구 (주문 취소) | notification (재입고 알림) |

## 수신 이벤트

| 이벤트 | 발행 모듈 | 처리 내용 |
|:---|:---|:---|
| `OrderCreatedEvent` | order | 주문 항목만큼 재고 차감 |
| `OrderCancelledEvent` | order | 차감한 재고 복구 |

## 비즈니스 규칙

- 상품명은 필수, 최대 100자
- 가격은 0 이상
- 재고 차감 시 재고가 부족하면 `InsufficientStockException` 발생
- 재고 차감은 동시성 제어 필요 (비관적 락 또는 분산 락)

## 테스트 실행

```bash
./gradlew test --tests "*.product.*"
```
