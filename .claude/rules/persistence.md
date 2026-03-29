---
description: 영속성 계층(Persistence Adapter) 규칙. JPA Entity, Repository, Mapper 작성 시 적용. 각 모듈은 자기 소유 테이블만 접근.
globs: src/main/java/**/adapter/out/persistence/**/*.java
---

# 영속성 계층 규칙

## 모듈별 테이블 소유권 (Spring Modulith 환경)

각 모듈은 자기 접두어의 테이블만 소유하고 접근한다.
**다른 모듈의 테이블에 직접 JOIN하거나 FK를 걸지 않는다.**

| 모듈 | 접두어 | 예시 테이블 |
|:---|:---|:---|
| order | `ord_` | `ord_order`, `ord_order_item` |
| product | `prd_` | `prd_product`, `prd_category` |
| member | `mbr_` | `mbr_member`, `mbr_address` |
| payment | `pay_` | `pay_payment`, `pay_refund` |
| delivery | `dlv_` | `dlv_delivery`, `dlv_tracking` |
| notification | `ntf_` | `ntf_notification` |

```java
// ✅ 같은 모듈 테이블 간 JOIN
@Query("SELECT o FROM OrderJpaEntity o JOIN FETCH o.items WHERE o.id = :id")
Optional<OrderJpaEntity> findByIdWithItems(@Param("id") Long id);

// ❌ 금지: 다른 모듈 테이블 직접 JOIN
@Query("SELECT o FROM OrderJpaEntity o JOIN ProductJpaEntity p ON ...")  // ❌
```

다른 모듈의 데이터가 필요하면 공개 API(`api/` 인터페이스)를 통해 조회하거나 이벤트로 데이터를 동기화한다.

### 모듈별 Flyway 마이그레이션 분리

```
src/main/resources/db/migration/
├── order/V2026032901__create_order_tables.sql        ← team-order만 수정
├── product/V2026032901__create_product_tables.sql    ← team-product만 수정
├── member/V2026032901__create_member_tables.sql      ← team-member만 수정
└── payment/V2026032901__create_payment_tables.sql    ← team-payment만 수정
```

## Domain Model ↔ JPA Entity 완전 분리

헥사고날 아키텍처의 핵심은 Domain이 영속성 기술(JPA)에 의존하지 않는 것이다.
Domain Model과 JPA Entity는 반드시 별도 클래스로 유지한다.

```
Domain Model (domain/model/)     ←→     JPA Entity (adapter/out/persistence/entity/)
       Order.java                              OrderJpaEntity.java
       OrderItem.java                          OrderItemJpaEntity.java
                          ↕
              OrderPersistenceMapper.java (MapStruct)
```

## JPA Entity 규칙

```java
@Entity
@Table(name = "ord_order")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderJpaEntity extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "order_id")
    private Long id;

    @Column(name = "member_id", nullable = false)
    private Long memberId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private OrderStatus status;

    @Column(name = "total_amount", nullable = false)
    private Long totalAmount;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItemJpaEntity> items = new ArrayList<>();
}
```

### Entity 네이밍
- 클래스: `{Domain}JpaEntity`
- 테이블: `{모듈접두어}_{테이블명}` (예: `ord_order`, `prd_product`)
- 컬럼: snake_case
- 외래키: `{참조테이블}_id`

### Entity 어노테이션
- `@NoArgsConstructor(access = PROTECTED)`: JPA 필수, 외부 생성 차단
- `@Getter`: 허용 (Mapper에서 필요)
- `@Setter`: 금지 (변경은 JPA 더티체킹 또는 명시적 메서드)
- `@Builder`: Entity에 직접 사용 금지 (Mapper 또는 정적 팩토리 사용)

### BaseTimeEntity

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
}
```

## MapStruct Mapper

```java
@Mapper(componentModel = "spring")
public interface OrderPersistenceMapper {

    // Domain → JPA Entity
    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "memberId.value", target = "memberId")
    @Mapping(source = "totalAmount.value", target = "totalAmount")
    OrderJpaEntity toJpaEntity(Order order);

    // JPA Entity → Domain
    @Mapping(target = "id", expression = "java(OrderId.of(entity.getId()))")
    @Mapping(target = "memberId", expression = "java(MemberId.of(entity.getMemberId()))")
    @Mapping(target = "totalAmount", expression = "java(Money.of(entity.getTotalAmount()))")
    Order toDomain(OrderJpaEntity entity);

    List<Order> toDomainList(List<OrderJpaEntity> entities);
}
```

### Mapper 규칙
- MapStruct `@Mapper(componentModel = "spring")` 사용
- Value Object (ID, Money 등) 변환은 `expression`으로 명시
- 복잡한 후처리: `@AfterMapping` 메서드 활용
- 변경 후 반드시 `./gradlew clean build` (annotation processor 캐시)

## Persistence Adapter (Port 구현체)

```java
@Repository
@RequiredArgsConstructor
class OrderPersistenceAdapter implements SaveOrderPort, LoadOrderPort {

    private final OrderJpaRepository orderJpaRepository;
    private final OrderPersistenceMapper mapper;

    @Override
    public OrderId save(Order order) {
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        OrderJpaEntity saved = orderJpaRepository.save(entity);
        return OrderId.of(saved.getId());
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return orderJpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public Page<Order> findByMemberId(MemberId memberId, Pageable pageable) {
        return orderJpaRepository.findByMemberId(memberId.value(), pageable)
            .map(mapper::toDomain);
    }
}
```

### Adapter 규칙
- 접근 제한자: `class` (package-private) — 외부 모듈에서 직접 접근 불가
- `@Repository` 어노테이션으로 예외 변환 AOP 적용
- 하나의 Adapter가 여러 Outbound Port를 구현 가능

## Spring Data Repository

```java
// package-private: 외부에서 직접 접근 금지
interface OrderJpaRepository extends JpaRepository<OrderJpaEntity, Long> {

    Page<OrderJpaEntity> findByMemberId(Long memberId, Pageable pageable);

    @Query("SELECT o FROM OrderJpaEntity o JOIN FETCH o.items WHERE o.id = :id")
    Optional<OrderJpaEntity> findByIdWithItems(@Param("id") Long id);
}
```

### Repository 규칙
- `interface` 접근 제한자: package-private (public 금지)
- 외부에서는 반드시 `PersistenceAdapter`를 통해서만 접근
- 복잡한 동적 쿼리: QueryDSL `{Domain}QueryRepository` 별도 분리
- N+1 문제: `JOIN FETCH` 또는 `@EntityGraph`로 해결
- 대량 조회: Projection(Interface/DTO) 활용하여 필요한 컬럼만 조회

## QueryDSL (복잡한 조회)

```java
@Repository
@RequiredArgsConstructor
class OrderQueryRepository {

    private final JPAQueryFactory queryFactory;

    public Page<OrderJpaEntity> search(OrderSearchCondition condition, Pageable pageable) {
        List<OrderJpaEntity> content = queryFactory
            .selectFrom(orderJpaEntity)
            .where(
                memberIdEq(condition.memberId()),
                statusIn(condition.statuses()),
                createdAtBetween(condition.from(), condition.to())
            )
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .orderBy(orderJpaEntity.createdAt.desc())
            .fetch();

        JPAQuery<Long> countQuery = queryFactory
            .select(orderJpaEntity.count())
            .from(orderJpaEntity)
            .where(
                memberIdEq(condition.memberId()),
                statusIn(condition.statuses()),
                createdAtBetween(condition.from(), condition.to())
            );

        return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchOne);
    }

    // BooleanExpression 메서드는 null 반환 시 자동으로 조건 무시
    private BooleanExpression memberIdEq(Long memberId) {
        return memberId != null ? orderJpaEntity.memberId.eq(memberId) : null;
    }
}
```
