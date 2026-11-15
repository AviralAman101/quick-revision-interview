# Spring Boot Event-Driven Patterns: Kafka Interview Revision

## 1. Publishing and consuming events

A producer publishes an event to a Kafka topic. A consumer service listens to that topic and processes the event. Use a stable message key, such as `orderId`, when events for the same aggregate need to remain ordered. Treat event payloads as versioned contracts.

```java
// Order service
@Transactional
Order createOrder(CreateOrder request) {
    Order order = repository.save(toOrder(request));
    kafkaTemplate.send("order-created", order.id().toString(),
                       new OrderCreated(order.id(), order.customerId()));
    return order;
}

// Consumer service
@KafkaListener(topics = "order-created", groupId = "billing")
void onOrderCreated(OrderCreated event) {
    billingService.startBilling(event.orderId());
}
```

**Interview: What does a consumer group do?**  Consumers within one group share partitions, so each partition is assigned to at most one consumer in that group at a time. Different groups each receive their own copy of the topic's events. A group cannot actively consume more partitions concurrently than the topic has.

## 2. Reliable publishing: the outbox pattern

A database transaction generally cannot atomically commit both a database write and a Kafka publish. The outbox pattern writes the business record and an event record to the same database transaction. A separate publisher sends pending outbox records to Kafka and marks them published. It may publish a record more than once, so consumers must be idempotent.

```java
@Transactional
Order createOrder(CreateOrder request) {
    Order order = repository.save(toOrder(request));
    outboxRepository.save(new OutboxEvent(UUID.randomUUID(),
                                          "OrderCreated", serialize(order)));
    return order;
}

void publishPendingEvents() {
    for (OutboxEvent event : outboxRepository.findPendingBatch()) {
        kafkaTemplate.send("order-created", event.id().toString(), event.payload())
            .whenComplete((result, error) -> {
                if (error == null) outboxRepository.markPublished(event.id());
            });
    }
}
```

**Interview: What are the trade-offs?**  The outbox prevents losing the intent to publish when the business transaction commits. It adds storage, cleanup, monitoring, and publisher complexity. Polling is straightforward but adds database load; CDC can avoid polling but needs additional infrastructure.

## 3. Retries and dead-letter topics

Use bounded retries with backoff for transient failures. Route messages that still fail, or that are permanently invalid, to a dead-letter topic (DLT). Monitor the DLT and define a safe investigation and replay process. Retries can repeat side effects and delay work, so consumers should be idempotent.

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltTopicSuffix = ".dlt")
@KafkaListener(topics = "order-created")
void onOrderCreated(OrderCreated event) {
    billingService.startBilling(event.orderId());
}

@DltHandler
void onDeadLetter(OrderCreated event) {
    alerting.recordFailure(event);
}
```

**Interview: What should you consider before retrying?**  Classify transient versus permanent errors, bound attempts, use backoff, avoid retry storms, account for ordering, and protect side effects from duplicates. A DLT does not fix the error by itself.

## 4. Idempotent consumers

Assume a message can be delivered more than once. Give each event a stable ID and store processed IDs, or enforce uniqueness through a database constraint/business key. Commit the business change and processed-event record in one database transaction.

```java
@Transactional
void onPaymentRequested(PaymentRequested event) {
    if (processedEvents.exists(event.eventId())) return;
    paymentService.createPayment(event); // also enforce a unique business/event key
    processedEvents.save(event.eventId());
}
```

**Interview: Does Kafka exactly-once remove the need for idempotency?**  No. Kafka transactions can provide exactly-once guarantees for certain Kafka-to-Kafka flows, but they do not automatically make arbitrary database writes or external API calls exactly once.

## 5. Ordering, keys, and partitions

Kafka preserves record order within a partition, not across a whole topic. Using the same key, such as `orderId`, sends related records to the same partition, preserving their relative order there. Increasing partition count can alter key-to-partition mapping, so consider ordering requirements when changing it.

```java
kafkaTemplate.send("order-events", event.orderId().toString(), event);
```

**Interview: How do you choose partition count?**  Estimate throughput and desired parallelism, while considering broker capacity and operational overhead. A consumer group can process at most one consumer per partition concurrently. Too few partitions limit parallel work; too many increase resource and management costs.

## 6. Event-driven workflows: sagas

A saga coordinates local transactions across services without one distributed database transaction. Each service commits its own work and publishes an event. Later failures trigger compensating actions, such as refunding a payment or releasing a reservation.

```text
Order service:     save order -> publish OrderCreated
Payment service:   OrderCreated -> charge -> PaymentSucceeded / PaymentFailed
Inventory service: PaymentSucceeded -> reserve -> StockReserved / StockUnavailable
Order service:     StockReserved -> confirm
                   StockUnavailable -> cancel and request refund
```

**Interview: Choreography or orchestration?**  Choreography lets services react to events and can fit small, clear workflows. As branching, timeouts, and compensations grow, an orchestrator can make the process easier to understand and operate, at the cost of introducing a coordinator.

## 7. Scaling and operating consumers

`@KafkaListener` settings:

- `topics` names the topic(s) the listener reads.
- `groupId` identifies the consumer group. Consumers in the same group share partition assignments; different groups each receive the events independently.
- `concurrency` tells Spring Kafka how many consumer threads to start for that listener in each application instance. It does **not** create partitions or set the topic's partition count.

```java
@KafkaListener(topics = "order-created", groupId = "billing", concurrency = "3")
void onOrderCreated(OrderCreated event) {
    billingService.startBilling(event.orderId());
}
```

If the topic has 6 partitions, concurrency 3 creates 3 consumers that may receive about 2 partitions each. Concurrency 6 can use all 6 partitions. Concurrency 8 leaves at least 2 consumers idle. If 2 application instances each run concurrency 3, the group has up to 6 consumers; with 6 partitions, all may be active. The exact assignment is managed by Kafka and depends on the assignor and group membership.

**Microservices clarification:**  This does not contradict microservice architecture. The partition limit caps parallel consumption within one consumer group for a topic. Independent services can subscribe using different group IDs and each receive all events. To increase parallelism for one group, add partitions when suitable; adding consumers beyond the partition count alone will not help.

**Interview: What is consumer lag?**  Lag is the distance between the latest topic offsets and the offsets processed by a consumer group. It can grow when input exceeds processing capacity, consumers block on dependencies, partition parallelism is insufficient, keys distribute unevenly, or rebalances interrupt work. Monitor sustained lag and diagnose its cause.

**Interview: How do you evolve event schemas?**  Prefer compatible changes, such as adding optional fields with defaults. Avoid changing the meaning or type of existing fields in place. Use a schema strategy or registry as appropriate, and roll out producers and consumers so old and new versions can coexist during deployment.

## Quick mental model

```text
Topic partition count = upper bound on active consumer parallelism per group
Listener concurrency = consumer threads per application instance for that listener
Application instances × listener concurrency = potential consumers in that group
Consumer groups = independent subscriptions, each receiving its own event stream
```

## Topics to cover next

- Rate limiting: fixed window, sliding window, token bucket, per-user/IP limits, Redis-backed distributed limits, and HTTP 429 handling
- Caching and cache invalidation
- Circuit breakers, timeouts, and bulkheads
- Spring Security and authentication/authorization
- Database transactions and API design
- Observability: logs, metrics, traces, and health checks
- AWS event-driven equivalents: SNS, SQS, EventBridge, and Step Functions

---

# Spring Boot Interview Revision: Resilience4j

## What is Resilience4j?

Resilience4j is a lightweight fault-tolerance library that can wrap calls with patterns such as Circuit Breaker, Retry, Rate Limiter, Bulkhead, and Time Limiter. In Spring Boot, the Spring Boot starter integrates annotations and configuration properties. Pick only the patterns that address a real failure mode; stacking decorators changes behavior and should be intentional.

## Circuit Breaker: states

**Interview: What are the Circuit Breaker states?**

- `CLOSED`: calls are allowed; outcomes are recorded. When configured failure or slow-call thresholds are exceeded, the breaker opens.
- `OPEN`: calls are rejected quickly with `CallNotPermittedException`; after the configured wait, it moves to `HALF_OPEN` (or transitions according to configured automatic behavior).
- `HALF_OPEN`: only a limited number of probe calls are allowed. Their outcomes decide whether the breaker closes or opens again.
- `METRICS_ONLY`: calls are allowed and metrics/events are recorded, but threshold breaches do not open the breaker.
- `DISABLED`: calls are allowed; breaker functionality is disabled.
- `FORCED_OPEN`: calls are rejected; the breaker is forced open.

The three primary operational states are CLOSED, OPEN, and HALF_OPEN. The other three are special states useful for monitoring or administrative control.

## Circuit Breaker: configuration

**Interview: What can you configure on a Circuit Breaker?**

Common settings include:

- `slidingWindowType`: `COUNT_BASED` or `TIME_BASED`.
- `slidingWindowSize`: number of calls or time buckets represented by the window.
- `minimumNumberOfCalls`: minimum outcomes before failure rate is calculated.
- `failureRateThreshold`: failure percentage that opens the breaker.
- `slowCallDurationThreshold` and `slowCallRateThreshold`: duration that counts as slow and the percentage that opens the breaker.
- `waitDurationInOpenState`: time before attempting recovery probes.
- `permittedNumberOfCallsInHalfOpenState`: number of probe calls.
- `automaticTransitionFromOpenToHalfOpenEnabled` and `maxWaitDurationInHalfOpenState`.
- Exception recording/ignoring predicates and event/metrics configuration.

A sliding window does not limit concurrency. It aggregates recent call outcomes; use a Bulkhead to cap concurrent calls.

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        slowCallDurationThreshold: 2s
        slowCallRateThreshold: 60
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        registerHealthIndicator: true
```

## Applying a Circuit Breaker

```java
@Service
class InventoryClient {
    @CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
    Inventory getInventory(String sku) {
        return httpClient.getInventory(sku);
    }

    Inventory fallback(String sku, CallNotPermittedException ex) {
        return Inventory.unknown(sku); // only if this is a safe business fallback
    }
}
```

A fallback should have a compatible signature and should distinguish breaker rejection from underlying business errors where needed. Do not turn failures into apparently valid success responses unless the fallback semantics are safe.

## Retry in Spring Boot / Resilience4j

**Interview: How does retry work?**

Retry invokes an operation again after a configured failure or retryable result, up to a bounded number of attempts, optionally waiting between attempts. Configure which exceptions/results are retried and which are ignored. Defaults and exact property names can vary by library version, so use explicit settings.

```yaml
resilience4j:
  retry:
    instances:
      catalog:
        maxAttempts: 3       # total attempts, including the first call
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
        ignoreExceptions:
          - com.example.InvalidRequestException
```

```java
@Retry(name = "catalog")
Product getProduct(String id) {
    return catalogClient.fetch(id);
}
```

Resilience4j retry can also be configured programmatically with `RetryConfig`/`RetryRegistry`, including exception predicates, ignored exceptions, result predicates, and custom interval calculations.

**Interview: Does `maxAttempts = 3` mean three retries?**

Usually it means three total invocations: the initial call plus up to two retries. Verify the API/version semantics when configuring it; Resilience4j documents `maxAttempts` as including the initial call.

## Retry vs Circuit Breaker

- Retry handles likely transient failures by trying again.
- Circuit Breaker stops calls to a dependency that is failing or too slow, giving it time to recover.
- Retry can amplify load during an outage. Bound attempts, use backoff (often with jitter), and avoid retrying non-idempotent operations unless they have idempotency protection.
- Their composition order matters: retrying outside a breaker can cause each attempt to pass through the breaker; a breaker outside retry can record the overall retried operation as one call. Configure deliberately and know the integration's decorator order.

## Resilience4j Retry vs Spring Retry

Both provide retry behavior but are separate libraries with different integrations and APIs. Resilience4j is part of a broader resilience toolkit and supports annotation/configuration based retry alongside circuit breakers, rate limiters, and bulkheads. Spring Retry offers Spring-oriented retry and recovery features. Avoid enabling both on the same method unintentionally, or attempts may multiply (for example, 3 attempts at each layer can yield up to 9 calls).

## Retry caveats

1. Retry only transient, recoverable failures; don't retry validation or authorization errors.
2. Set a maximum attempt count and backoff. Add jitter in high fan-out systems to reduce synchronized retries.
3. Ensure the operation is idempotent or use an idempotency key, especially for writes and payment-like calls.
4. Fit total attempts and waits within the caller's end-to-end timeout/deadline.
5. A fallback is a deliberate degradation path, not a generic way to hide failures.
6. Observe retry count, final failures, breaker state, latency, and dependency health.

## Interview quick answers

**What happens when an OPEN breaker receives a call?**  It short-circuits without invoking the dependency and typically throws `CallNotPermittedException`; application fallback handling may then run.

**Why is `minimumNumberOfCalls` important?**  It prevents the breaker from making a decision from too few observations. For example, one failure out of one call should not necessarily open a breaker configured to evaluate a larger sample.

**Does a Circuit Breaker cancel an in-flight slow call?**  No. It measures call duration and can reject future calls after thresholds are met; it does not itself terminate an already-running call. Use client timeouts and, if appropriate, a Time Limiter.

**Should every downstream call use retry and circuit breaker?**  Apply them according to the failure mode and operation semantics. Retries can worsen overload; breakers can hide a recovery path if misconfigured. Set timeouts at the network client too.

## Sources

- [Resilience4j CircuitBreaker documentation](https://resilience4j.readme.io/docs/circuitbreaker)
- [Resilience4j Retry documentation](https://resilience4j.readme.io/docs/retry)
- [Resilience4j getting started](https://resilience4j.readme.io/docs/getting-started)
