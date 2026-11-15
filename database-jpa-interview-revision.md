# Database, JPA, and Transaction Interview Revision

## SQL and database fundamentals

### 1. What is the difference between `INNER JOIN` and `LEFT JOIN`?

An `INNER JOIN` returns rows where the join condition matches in both tables. A `LEFT JOIN` returns every row from the left table plus matches from the right; unmatched right-side columns are `NULL`.

```sql
-- Only customers with orders
SELECT c.id, o.id
FROM customer c
INNER JOIN orders o ON o.customer_id = c.id;

-- All customers, including those without orders
SELECT c.id, o.id
FROM customer c
LEFT JOIN orders o ON o.customer_id = c.id;
```

### 2. What are `RIGHT JOIN` and `FULL OUTER JOIN`?

A `RIGHT JOIN` preserves every row from the right table plus matching rows from the left. A `FULL OUTER JOIN` preserves rows from both sides, matching them where possible and filling unmatched columns with `NULL`. Database support for `FULL OUTER JOIN` varies.

### 3. What is the difference between a join and a subquery?

A join combines related rows from tables. A subquery is a query nested inside another query. Either can express similar logic; performance depends on the database, indexes, data volume, and execution plan. Prefer the clearer form, and inspect the plan when performance matters.

### 4. What is a `UNION`? How does it differ from `UNION ALL`?

Both combine results from compatible `SELECT` statements with the same number of columns and compatible types. `UNION` removes duplicate rows, which can require extra work. `UNION ALL` retains duplicates and is usually preferable when deduplication is not required.

```sql
SELECT email FROM customers
UNION ALL
SELECT email FROM suppliers;
```

### 5. What does ACID mean?

- **Atomicity:** A transaction succeeds as a unit or rolls back.
- **Consistency:** It preserves defined data rules and constraints.
- **Isolation:** Concurrent work is controlled according to the selected isolation level.
- **Durability:** Committed changes survive failures, subject to database durability guarantees.

### 6. What are common transaction isolation levels?

- **Read Uncommitted:** May allow dirty reads.
- **Read Committed:** Prevents dirty reads; repeated reads may see newer committed data.
- **Repeatable Read:** Protects rows already read from certain concurrent changes; phantom behavior depends on database and implementation.
- **Serializable:** Strongest isolation; transactions behave as if run one at a time, often reducing concurrency.

Exact guarantees vary between database engines.

### 7. What is optimistic versus pessimistic locking in JPA?

Optimistic locking assumes conflicts are uncommon and detects them when changes are written, usually using a version field. Pessimistic locking asks the database to lock rows so competing transactions wait or fail.

```java
@Entity
class Account {
    @Id Long id;
    @Version Long version;
}
```

With `@Version`, a concurrent update is detected rather than silently overwriting another transaction's change.

### 8. What does `@Transactional` do in Spring?

It lets Spring manage a transaction around a method: successful completion commits, while failures roll back according to rollback rules. By default, unchecked exceptions and errors trigger rollback; checked exceptions generally do not unless configured. With proxy-based transaction management, the method usually must be called through the Spring proxy for the transaction advice to apply.

### 9. What is the N+1 query problem?

A query loads a list of entities, then an additional query runs for each entity to load a relationship. This creates many database round trips. Remedies include fetch joins, entity graphs, batch fetching, or projections. Avoid making every relationship eager, which can load excessive data.

```jpql
SELECT o FROM Order o
JOIN FETCH o.customer
WHERE o.status = :status
```

### 10. What is the difference between lazy and eager loading?

Lazy loading fetches a relationship when accessed; eager loading fetches it with or soon after the initial entity load. Lazy loading can reduce unnecessary reads but may cause N+1 queries or fail outside an active persistence context. Eager loading can retrieve more data than needed. Define fetch plans for the use case.

## More JPA and transaction questions

### 11. What is the persistence context, and how does dirty checking work?

The persistence context is JPA's first-level cache and unit of managed entities. When a managed entity changes, the provider detects the difference (dirty checking) and writes an update during flush or transaction commit. Calling `save` is not always required for a managed entity's changes to be persisted.

```java
@Transactional
void renameCustomer(long id, String name) {
    Customer customer = entityManager.find(Customer.class, id);
    customer.setName(name); // dirty checking persists this at flush/commit
}
```

### 12. What is the difference between `persist`, `merge`, and Spring Data `save`?

`persist` makes a new entity managed. `merge` copies state into a managed instance and returns that managed instance; the passed object itself may remain detached. Spring Data JPA's `save` chooses `persist` or `merge` based on whether it considers the entity new. Use the returned value from `save`, especially when merging detached objects.

### 13. What are entity lifecycle states?

- **Transient/new:** Not associated with a persistence context and not yet persisted.
- **Managed:** Associated with the current persistence context; changes are tracked.
- **Detached:** Was managed, but is no longer attached; changes are not automatically tracked.
- **Removed:** Marked for deletion, typically executed on flush or commit.

### 14. What is the difference between `flush` and `commit`?

`flush` synchronizes pending persistence-context changes to the database, but does not commit the transaction. The transaction can still roll back afterward. `commit` completes the transaction and makes its changes durable according to database guarantees. JPA may flush automatically before certain queries or at commit.

### 15. What are `CascadeType` and `orphanRemoval`?

Cascade propagates selected entity operations (such as persist or remove) from a parent to related entities. `orphanRemoval = true` deletes a child when it is removed from the parent's relationship. Use them when the child lifecycle is truly owned by the parent; cascading removal across shared entities can cause accidental data loss.

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL,
           orphanRemoval = true)
private List<OrderLine> lines;
```

### 16. What is the owning side of a JPA relationship?

The owning side controls the foreign-key update. In a bidirectional relationship, `mappedBy` marks the inverse side; the other side owns the relationship. Update both sides in application code to keep the in-memory object graph consistent, but ensure the owning side is set for persistence.

```java
// Order is inverse side
@OneToMany(mappedBy = "order")
private List<OrderLine> lines;

// OrderLine owns the foreign key
@ManyToOne
@JoinColumn(name = "order_id")
private Order order;
```

### 17. How does transaction propagation work? What is `REQUIRED` versus `REQUIRES_NEW`?

`REQUIRED` joins an existing transaction or starts one if none exists; it is the common default. `REQUIRES_NEW` suspends an existing transaction and starts an independent one. The new transaction can commit even if the outer transaction later rolls back, so use it only when that independent durability is intended.

### 18. What is transaction isolation, and how do you set it in Spring?

Isolation determines which effects of concurrent transactions are visible. Spring can request an isolation level on `@Transactional`, but the database determines the actual semantics and may not support every level identically.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
void updateInventory(...) { ... }
```

Choose isolation based on the consistency requirement and concurrency cost; also consider constraints and locking.

### 19. When does Spring roll back a transaction, and how do you change the rule?

By default, Spring rolls back for unchecked exceptions (`RuntimeException`) and `Error`, not checked exceptions. Configure `rollbackFor` or `noRollbackFor` when the default does not fit the business behavior. Catching an exception and returning normally can lead to commit unless the transaction is explicitly marked rollback-only.

```java
@Transactional(rollbackFor = BusinessCheckedException.class)
void performOperation() throws BusinessCheckedException { ... }
```

### 20. Why might `@Transactional` not work on a method?

Common causes include self-invocation bypassing the Spring proxy, calling a method on an object not managed by Spring, incorrect transaction-manager configuration, or relying on proxying behavior that does not apply to the method/class shape. Also check exception and rollback rules. Put the transaction boundary on a Spring-managed service method invoked through its proxy.

## Revision reminders

- SQL join semantics and JPA relationship mappings are related, but not the same thing: a JPA association is an object model mapping; SQL joins are query operations.
- Avoid assuming that lazy loading is always safe or that eager loading is always efficient.
- Use database constraints for invariants that must hold under concurrency; application checks alone can race.
- State the database engine when discussing exact isolation, locking, or SQL syntax behavior.
