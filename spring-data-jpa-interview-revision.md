# Spring Data JPA Interview Questions and Answers

## 1. What does a Spring Data repository do?

A repository is an interface that Spring implements at runtime. It provides common persistence operations and can derive queries from method names or run declared queries.

```java
interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByEmail(String email);
}
```

## 2. Which repository interfaces are available, and when should you use each?

- `Repository<T, ID>` is a marker interface. Use it when you want to expose only selected methods.
- `CrudRepository<T, ID>` provides common CRUD operations.
- `PagingAndSortingRepository<T, ID>` adds paging and sorting methods.
- `JpaRepository<T, ID>` combines common CRUD and paging capabilities with JPA-specific operations such as flushing and batch methods.

In modern Spring Data versions, `PagingAndSortingRepository` and `CrudRepository` are separate interfaces; older versions had a different inheritance structure. Check the version used by the project.

## 3. How do you implement pagination?

Pass a `Pageable` parameter and return a `Page<T>` when total counts and page counts are required.

```java
interface OrderRepository extends JpaRepository<Order, Long> {
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
}

Pageable pageable = PageRequest.of(
    pageNumber, // zero-based
    pageSize,
    Sort.by("createdAt").descending()
        .and(Sort.by("id").descending()) // stable tie-breaker
);

Page<Order> result = orderRepository.findByCustomerId(customerId, pageable);
```

A typical workflow is: controller accepts page/size/sort, service validates and creates `Pageable`, repository fetches that page, and API returns the items and requested metadata. A stable tie-breaker avoids unpredictable movement among rows with equal sort values.

## 4. What is the difference between `Page`, `Slice`, and `List` with `Pageable`?

- `Page<T>` includes total elements and total pages. A count query may be needed.
- `Slice<T>` tells whether another slice exists without requiring a total count.
- `List<T>` with `Pageable` returns only the requested range, without paging metadata or a count query.

Use `Page` when the UI needs “page X of Y”; use `Slice` for “load more” when a total is unnecessary.

## 5. What are the limitations of offset pagination?

Deep offsets can become expensive because the database may need to walk past many rows. Inserts or deletes can also shift results between page requests. For large changing datasets, consider keyset scrolling: sort by stable indexed fields and request rows after the last seen key. Keyset scrolling is efficient for sequential navigation but does not naturally provide total page counts.

## 6. When should you use a derived query method?

Use derived methods for short, clear predicates. Spring parses the method name and creates the query.

```java
List<Order> findByStatusAndCreatedAtAfter(OrderStatus status, Instant after);
Optional<Customer> findByEmailIgnoreCase(String email);
boolean existsByCustomerIdAndStatus(Long id, OrderStatus status);
List<Product> findTop5ByCategoryOrderByPriceAsc(String category);
```

If the method name becomes long or hard to understand, use `@Query` or a specification.

## 7. When should you use `@Query`?

Use `@Query` when JPQL is clearer than a method name, or when you need a custom projection or specific query shape. JPQL uses entities and properties; native queries use the database SQL dialect.

```java
@Query("""
    select o from Order o
    where o.customer.id = :customerId
      and o.status = :status
    order by o.createdAt desc
    """)
List<Order> findRecentOrders(Long customerId, OrderStatus status);
```

For complex paginated queries, provide a matching count query if Spring cannot derive the right one or the derived query is inefficient.

## 8. How do you implement a search with optional filters?

For a few fixed combinations, derived methods may work. For many optional criteria, use `Specification` or Criteria API so predicates can be composed.

```java
interface OrderRepository extends
        JpaRepository<Order, Long>,
        JpaSpecificationExecutor<Order> {
}

Specification<Order> spec = Specification
    .where(hasStatus(status))
    .and(createdAfter(fromDate))
    .and(belongsToCustomer(customerId));

Page<Order> page = orderRepository.findAll(spec, pageable);
```

## 9. How do you update or delete rows with a custom query?

Use `@Modifying` and execute it in a transaction. Bulk updates operate directly on database rows, so entities already loaded in the persistence context can become stale.

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("update Order o set o.status = :status where o.id = :id")
int updateStatus(Long id, OrderStatus status);
```

```java
@Transactional
int cancelOrder(Long id) {
    return repository.updateStatus(id, OrderStatus.CANCELLED);
}
```

Bulk operations bypass normal per-entity lifecycle behavior and dirty checking.

## 10. What is a projection, and when is it useful?

A projection selects only fields needed by a use case instead of loading a full entity graph. It can reduce data transfer and unnecessary entity loading.

```java
interface OrderSummary {
    Long getId();
    Instant getCreatedAt();
    BigDecimal getTotal();
}

List<OrderSummary> findByCustomerId(Long customerId);
```

For a DTO projection:

```java
@Query("""
    select new com.example.OrderSummaryDto(o.id, o.createdAt, o.total)
    from Order o where o.customer.id = :customerId
    """)
List<OrderSummaryDto> findSummaries(Long customerId);
```

Use projections for read-focused views. Use entities when business logic needs to modify them and rely on JPA tracking.

## 11. How do you avoid N+1 queries from repository methods?

Choose a fetch plan for each use case. Options include fetch joins, `@EntityGraph`, batch fetching, or a projection.

```java
@EntityGraph(attributePaths = {"customer"})
Page<Order> findByStatus(OrderStatus status, Pageable pageable);
```

Be careful with collection fetch joins and pagination: they can multiply rows and make paging inefficient or incorrect. A two-step query or projection may be better.

## 12. How do you check existence without loading an entity?

Use an `existsBy...` derived method:

```java
boolean existsByEmailIgnoreCase(String email);
```

For uniqueness guarantees, also add a database unique constraint. An application-level existence check can race with another transaction.

## 13. How should a repository return one result that may not exist?

Use `Optional<T>` when the query should return zero or one row. If multiple rows match, Spring Data generally reports a non-unique result instead of silently selecting one.

```java
Optional<Customer> findByEmail(String email);
```

Enforce business uniqueness with a database constraint too.

## 14. What is `@EntityGraph`, and how does it differ from changing fetch type?

`@EntityGraph` specifies relationships to fetch for a particular repository operation, rather than making them eager for every use of the entity.

```java
@EntityGraph(attributePaths = {"customer", "items"})
Optional<Order> findWithDetailsById(Long id);
```

It is a per-query fetch plan. Check generated SQL for performance-sensitive paths because it does not guarantee one SQL query in every case.

## 15. How do you apply a pessimistic lock through Spring Data JPA?

Use `@Lock` on a repository query and call it inside a transaction.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(Long id);
```

Lock behavior and timeout support depend on the database. Use it for specific contention-sensitive operations, not every read.

## 16. What is a repository fragment?

A repository fragment adds custom behavior through a separate interface and implementation. Use it when derived queries, `@Query`, or specifications do not fit, such as specialized Criteria API logic or database-specific behavior.

```java
interface OrderSearch {
    List<Order> searchComplexCriteria(OrderFilter filter);
}

interface OrderRepository extends JpaRepository<Order, Long>, OrderSearch {}
```

## 17. Does `save` always insert a row?

No. Spring Data JPA determines whether the entity is new; it uses `persist` for new entities and `merge` for entities considered existing. With `merge`, the returned object is the managed instance, so use the return value when working with detached entities.

```java
Customer managedCustomer = repository.save(customer);
```

SQL may not execute until flush or commit.

## 18. What is an uncommon query derivation issue involving `findById`?

`findById` is a reserved repository method that targets the entity identifier, even if the identifier property is not literally named `id`. If the entity also has a business field called `id`, use a descriptive token to query that field:

```java
Optional<Customer> findCustomerById(String externalId);
```

## 19. What if Spring Data parses a nested property path incorrectly?

Property paths can be ambiguous. Use `_` to mark an explicit traversal boundary:

```java
List<Person> findByAddress_ZipCode(String zipCode);
```

This explicitly traverses `address.zipCode`, avoiding ambiguity with a direct property such as `addressZipCode`.

## 20. Can repository query methods return `Stream<T>`?

Yes, in supported cases. A stream may hold database resources such as a cursor, so close it promptly and consume it within the appropriate transaction/persistence context.

```java
@Transactional(readOnly = true)
void exportOrders() {
    try (Stream<Order> orders = repository.streamAllByStatus(OrderStatus.PAID)) {
        orders.forEach(exporter::write);
    }
}
```

A `Stream<T>` does not automatically guarantee memory-free processing; fetch behavior depends on the provider and query.

## Quick guidelines

- Use derived methods for short, obvious predicates.
- Use `@Query` when the query is clearer explicitly.
- Use `Page` when total-count metadata is required; use `Slice` when only “has next” matters.
- Use keyset scrolling for large result sets with a stable indexed sort.
- Use projections for read models that need selected fields.
- Use `Specification` for dynamically composed filters.
- Enforce invariants such as uniqueness with database constraints; an application check alone can race.
- Inspect generated SQL and query plans for performance-sensitive methods.

## Official references

- [Spring Data repository concepts](https://docs.spring.io/spring-data/jpa/reference/repositories/core-concepts.html)
- [Spring Data query methods](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html)
- [Spring Data JPA query methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)
- [Spring Data specifications](https://docs.spring.io/spring-data/jpa/reference/jpa/specifications.html)
- [Spring Data scrolling](https://docs.spring.io/spring-data/jpa/reference/repositories/scrolling.html)
