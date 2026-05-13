# [아키텍처 시리즈 06] Data Model / Repository Architecture — 실무에서 틀리기 쉬운 것들

> 이 시리즈는 백엔드 개발자가 실무에서 마주치는 10가지 아키텍처를 "왜 이렇게 하면 안 되는가" 중심으로 정리합니다.
> 예제 코드는 Java 17 + Spring Boot 3.x 기준입니다.

---

## 개요

Repository Architecture는 데이터 접근 로직을 Repository 패턴으로 추상화해 도메인과 DB 기술을 분리하는 구조다. Spring Data JPA를 쓰면 쉽게 적용할 수 있지만, 잘못 쓰면 오히려 성능 문제와 유지보수 문제가 동시에 발생한다.

**핵심 질문 세 가지:**
- 도메인 로직이 Repository 안에 섞여 있지 않은가?
- N+1 문제가 발생하고 있지 않은가?
- 복잡한 조회 쿼리를 어디서 처리하고 있는가?

---

## 안티패턴 1 — JpaRepository를 Service에서 직접 사용

`JpaRepository`를 Service에서 직접 의존하면 도메인이 JPA 기술에 종속된다.

```java
// ❌ 잘못된 예: Service가 JpaRepository를 직접 참조
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderJpaRepository orderJpaRepository; // JPA 구현체에 직접 의존

    public List<OrderResponse> findByUserId(Long userId) {
        return orderJpaRepository.findByUserId(userId) // JPA 메서드 직접 호출
                .stream()
                .map(OrderResponse::from)
                .toList();
    }
}
```

**왜 문제인가?**

- DB를 MyBatis로 교체하거나 테스트에서 인메모리 구현체로 교체할 때 Service도 수정해야 함
- `JpaRepository`가 제공하는 `flush()`, `saveAndFlush()` 같은 JPA 전용 메서드를 Service에서 직접 호출하는 실수가 생김

```java
// ✅ 올바른 예: 도메인 인터페이스로 추상화
// 도메인이 정의하는 Repository 인터페이스
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findByUserId(Long userId);
}

// Service는 인터페이스에만 의존
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository; // 인터페이스에 의존

    public List<OrderResponse> findByUserId(Long userId) {
        return orderRepository.findByUserId(userId)
                .stream()
                .map(OrderResponse::from)
                .toList();
    }
}

// JPA 구현체는 infrastructure 레이어에
@Repository
@RequiredArgsConstructor
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderJpaRepository jpa;

    @Override
    public List<Order> findByUserId(Long userId) {
        return jpa.findByUserId(userId)
                .stream()
                .map(OrderEntity::toDomain)
                .toList();
    }
}
```

---

## 안티패턴 2 — N+1 문제를 인지하지 못하고 사용

JPA를 쓸 때 가장 많이 발생하는 성능 문제다. 연관관계를 조회할 때 쿼리가 N+1번 실행된다.

```java
// ❌ 잘못된 예: N+1이 발생하는 코드
@Entity
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items; // 지연 로딩
}

@Service
public class OrderService {
    public List<OrderResponse> findAll() {
        List<Order> orders = orderRepository.findAll(); // 쿼리 1번
        return orders.stream()
                .map(order -> {
                    order.getItems(); // 주문마다 쿼리 1번씩 추가 실행 → N+1
                    return OrderResponse.from(order);
                })
                .toList();
        // 주문이 100건이면 총 101번 쿼리 실행
    }
}
```

```java
// ✅ 올바른 예 1: fetch join으로 한 번에 조회
public interface OrderJpaRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :userId")
    List<Order> findByUserIdWithItems(@Param("userId") Long userId);
}

// ✅ 올바른 예 2: QueryDSL fetch join
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {
    private final JPAQueryFactory queryFactory;

    public List<Order> findByUserIdWithItems(Long userId) {
        return queryFactory
                .selectFrom(order)
                .join(order.items, orderItem).fetchJoin() // fetch join으로 N+1 방지
                .where(order.userId.eq(userId))
                .fetch();
    }
}

// ✅ 올바른 예 3: @EntityGraph
public interface OrderJpaRepository extends JpaRepository<Order, Long> {

    @EntityGraph(attributePaths = {"items"}) // items를 즉시 로딩
    List<Order> findByUserId(Long userId);
}
```

**N+1 확인 방법:**

```yaml
# application.yml — 개발 환경에서 쿼리 로그 활성화
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE
```

---

## 안티패턴 3 — 단순 조회에도 Repository를 거쳐 Entity로 반환

조회 전용 API인데도 Entity를 반환하고 Service에서 변환하면 불필요한 객체 생성이 많아진다.

```java
// ❌ 잘못된 예: 조회 전용인데 Entity 전체를 가져와서 변환
@Service
public class OrderService {
    public List<OrderSummaryResponse> getOrderSummaries(Long userId) {
        return orderRepository.findByUserId(userId) // Entity 전체 조회
                .stream()
                .map(order -> new OrderSummaryResponse( // 필요한 필드만 추출
                        order.getId(),
                        order.getStatus(),
                        order.getTotalPrice()
                ))
                .toList();
        // OrderItem 등 연관 데이터도 전부 로딩됨
    }
}
```

```java
// ✅ 올바른 예: DTO 직접 조회 (Projection)
// 필요한 필드만 담은 조회 전용 DTO
public record OrderSummaryResponse(Long id, OrderStatus status, int totalPrice) {}

// JPQL DTO 직접 조회
public interface OrderJpaRepository extends JpaRepository<Order, Long> {

    @Query("SELECT new com.example.order.dto.OrderSummaryResponse(" +
           "o.id, o.status, o.totalPrice) " +
           "FROM Order o WHERE o.userId = :userId")
    List<OrderSummaryResponse> findSummariesByUserId(@Param("userId") Long userId);
}

// QueryDSL Projections
@Repository
public class OrderQueryRepository {
    public List<OrderSummaryResponse> findSummariesByUserId(Long userId) {
        return queryFactory
                .select(Projections.constructor(OrderSummaryResponse.class,
                        order.id,
                        order.status,
                        order.totalPrice))
                .from(order)
                .where(order.userId.eq(userId))
                .fetch();
    }
}
```

---

## 안티패턴 4 — Repository에 페이징 처리 없이 전체 조회

데이터가 적을 때는 괜찮지만, 데이터가 쌓이면 전체 조회는 OOM(OutOfMemoryError)을 일으킨다.

```java
// ❌ 잘못된 예: 전체 데이터를 한 번에 조회
@Service
public class OrderService {
    public List<OrderResponse> findAll() {
        return orderRepository.findAll() // 데이터가 100만 건이면?
                .stream()
                .map(OrderResponse::from)
                .toList();
    }
}
```

```java
// ✅ 올바른 예: Pageable로 페이징 처리
@Service
public class OrderService {
    public PageResponse<OrderResponse> findAll(Pageable pageable) {
        Page<Order> page = orderRepository.findAll(pageable);
        return PageResponse.of(page.map(OrderResponse::from));
    }
}

// Controller
@GetMapping("/orders")
public ApiResponse<PageResponse<OrderResponse>> getOrders(
        @PageableDefault(size = 20, sort = "createdAt",
                         direction = Sort.Direction.DESC) Pageable pageable) {
    return ApiResponse.ok(orderService.findAll(pageable));
}

// 대용량 배치 처리가 필요한 경우: Slice로 커서 기반 페이징
public interface OrderJpaRepository extends JpaRepository<Order, Long> {
    Slice<Order> findByCreatedAtAfter(LocalDateTime cursor, Pageable pageable);
}
```

---

## 안티패턴 5 — 복잡한 동적 쿼리를 메서드 이름으로 처리

Spring Data JPA의 메서드 이름 쿼리는 조건이 2~3개를 넘어가면 가독성이 급격히 떨어진다.

```java
// ❌ 잘못된 예: 메서드 이름으로 복잡한 동적 쿼리 처리
public interface OrderJpaRepository extends JpaRepository<Order, Long> {

    // 조건이 늘어날수록 메서드 이름이 끝없이 길어짐
    List<Order> findByUserIdAndStatusAndCreatedAtBetweenAndTotalPriceGreaterThan(
            Long userId,
            OrderStatus status,
            LocalDateTime from,
            LocalDateTime to,
            int minPrice
    );
    // 조건이 optional이면? → 메서드를 경우의 수만큼 만들어야 함
}
```

```java
// ✅ 올바른 예: QueryDSL로 동적 쿼리 처리
public record OrderSearchCondition(
        Long        userId,
        OrderStatus status,    // null이면 조건 미적용
        LocalDate   fromDate,  // null이면 조건 미적용
        LocalDate   toDate,
        Integer     minPrice
) {}

@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {
    private final JPAQueryFactory queryFactory;

    public List<Order> search(OrderSearchCondition cond) {
        return queryFactory
                .selectFrom(order)
                .where(
                    userIdEq(cond.userId()),
                    statusEq(cond.status()),
                    createdAtBetween(cond.fromDate(), cond.toDate()),
                    minPriceGoe(cond.minPrice())
                )
                .orderBy(order.createdAt.desc())
                .fetch();
    }

    // 각 조건을 null 안전하게 처리
    private BooleanExpression userIdEq(Long userId) {
        return userId != null ? order.userId.eq(userId) : null;
    }

    private BooleanExpression statusEq(OrderStatus status) {
        return status != null ? order.status.eq(status) : null;
    }

    private BooleanExpression createdAtBetween(LocalDate from, LocalDate to) {
        if (from == null || to == null) return null;
        return order.createdAt.between(
                from.atStartOfDay(), to.plusDays(1).atStartOfDay());
    }

    private BooleanExpression minPriceGoe(Integer minPrice) {
        return minPrice != null ? order.totalPrice.goe(minPrice) : null;
    }
}
```

---

## Repository 설계 요약

| 상황 | 권장 방법 |
|---|---|
| 단순 CRUD | Spring Data JPA 메서드 이름 쿼리 |
| 연관관계 조회 | fetch join 또는 `@EntityGraph` |
| 조회 전용 (성능 최적화) | DTO Projection (JPQL / QueryDSL) |
| 복잡한 동적 쿼리 | QueryDSL |
| 대용량 배치 처리 | `Slice` 또는 커서 기반 페이징 |

---
> JPA 사용 시 N+1 문제는 연관관계가 있는 엔티티를 조회할 때 발생합니다. 예를 들어 주문 목록 N건을 조회하면 쿼리 1번, 각 주문의 상품 목록을 가져오려면 쿼리가 N번 추가 실행되어 총 N+1번 쿼리가 나가는 문제입니다. fetch join이나 `@EntityGraph`로 연관 데이터를 한 번에 가져오는 방식으로 해결하고, 단순 조회라면 DTO Projection으로 필요한 필드만 직접 조회하는 방식을 씁니다.

> 복잡한 검색 조건이 있을 때는 QueryDSL을 사용합니다. Spring Data JPA의 메서드 이름 쿼리는 조건이 2~3개를 넘어가면 메서드 이름이 지나치게 길어지고, 조건이 optional인 경우 경우의 수만큼 메서드를 만들어야 합니다. QueryDSL은 각 조건을 null 체크해서 동적으로 조합할 수 있어서 복잡한 검색 조건 처리에 적합합니다.
