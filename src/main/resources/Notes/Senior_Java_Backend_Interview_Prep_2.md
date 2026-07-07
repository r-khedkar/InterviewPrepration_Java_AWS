# Senior Java Backend Engineer — Interview Prep
### Spring Boot · Microservices · Kafka · 10 Years Experience

This guide is built the way a panel would actually interview a senior candidate — not "define X" but "how did you use X, and what trade-offs did you make." Answer in that style: concept → real decision you made → trade-off you weighed.

---

## 1. Core Java & Concurrency

**Q1. With 10 years of Java, what changed the most about how you write code from Java 7 to Java 17/21?**
> Stream API and lambdas changed how collections get processed — less boilerplate, more declarative. `Optional` reduced null-pointer defensive code. `var` for local type inference. Records (Java 14+) replaced verbose DTOs/POJOs with immutable data carriers. Sealed classes and pattern matching (Java 17/21) make exhaustive type handling safer. Virtual threads (Project Loom, Java 21) are the biggest one for backend work — they let you write blocking-style code that scales like async, which matters a lot for high-throughput microservices without rewriting everything in reactive style.

**Q2. Explain the difference between `synchronized`, `ReentrantLock`, and `ConcurrentHashMap` — when do you pick each?**
> `synchronized` is simplest, JVM-managed, but coarse — you can't try-lock, interrupt, or have fairness. `ReentrantLock` gives you `tryLock()`, timeouts, interruptibility, and fairness policies — useful when you need non-blocking attempts or finer control, e.g., a connection pool checkout. `ConcurrentHashMap` avoids locking the whole map — it segments internally — so for read-heavy/write-light shared state (like a local cache) it outperforms wrapping a `HashMap` in synchronized blocks. I default to `ConcurrentHashMap` for shared mutable state and only reach for explicit locks when I need ordering guarantees across multiple operations.

**Q3. What's the difference between `CompletableFuture` and a plain `Future`? Have you used it in production?**
> Plain `Future.get()` blocks. `CompletableFuture` supports composition (`thenApply`, `thenCompose`, `thenCombine`), exception handling (`exceptionally`, `handle`), and non-blocking callbacks. I've used it to parallelize independent downstream calls — e.g., calling pricing service and inventory service concurrently, then combining results with `thenCombine` instead of sequential blocking calls, cutting p99 latency roughly in half for that endpoint.

**Q4. Explain `volatile`. Does it guarantee atomicity?**
> No — `volatile` guarantees visibility (writes are immediately visible to other threads) and ordering (prevents instruction reordering around it via memory barriers), but not atomicity. `count++` on a volatile field is still a race condition because it's read-modify-write. For atomic counters I use `AtomicInteger`/`AtomicLong`; for atomic compound state, `AtomicReference` with CAS loops or just a lock.

**Q5. Memory leaks in Java — how have you diagnosed one in production?**
> Common culprits: static collections that keep growing, listener/callback references not removed, ThreadLocal not cleared in pooled threads, unbounded caches. My process: heap dump via `jmap` or trigger one through monitoring, analyze with Eclipse MAT looking at the dominator tree, find what's retaining the most memory and trace the GC roots back. One real case: a `ThreadLocal` used for request-scoped tracing context wasn't cleared in a thread pool, so contexts accumulated across requests — fixed by clearing in a `finally` block.

---

## 2. Spring Boot

**Q6. Explain Spring Boot auto-configuration — how does it actually decide what beans to create?**
> `@EnableAutoConfiguration` triggers a scan of `spring.factories` (or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in newer Boot) for `AutoConfiguration` classes. Each one is gated by conditional annotations — `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty` — so Boot only activates configuration when the relevant classes are on the classpath and no user-defined bean already exists. That's why adding a starter dependency "just works" but you can always override by defining your own bean.

**Q7. Constructor injection vs field injection — what do you actually enforce on your teams?**
> Constructor injection, always, for mandatory dependencies. It makes dependencies explicit and final, enables immutability, fails fast at startup if something's missing, and makes unit testing trivial without reflection tricks like `@InjectMocks` quirks. Field injection hides dependencies and makes a class harder to test in isolation. The only place I'd allow field injection is in test classes themselves.

**Q8. How do you handle cross-cutting concerns (logging, security, retries) in Spring Boot?**
> AOP for cross-cutting logic that isn't business logic — e.g., an `@Around` advice for method-level timing/audit logging, or a custom annotation like `@RateLimited` resolved via an aspect. Spring Security filter chain for auth concerns. Resilience4j annotations (`@Retry`, `@CircuitBreaker`, `@RateLimiter`) for resiliency instead of hand-rolling retry loops — they're declarative and centrally configurable via properties.

**Q9. Walk me through what happens when a request hits a `@RestController` endpoint with `@Transactional` and it throws an exception halfway through.**
> Spring's transaction proxy wraps the actual bean — `@Transactional` works via a proxy (CGLIB if no interface, JDK dynamic proxy if there is one), so it only applies to calls coming through the proxy, not internal self-invocation. By default, only unchecked exceptions trigger rollback (`RuntimeException`/`Error`); checked exceptions don't unless you specify `rollbackFor`. If a checked exception is thrown and not configured, the transaction commits despite the exception — a classic bug. I always make this explicit with `@Transactional(rollbackFor = Exception.class)` when checked exceptions are in play, or design services to throw unchecked exceptions for failure paths.

**Q10. Have you hit the self-invocation problem with `@Transactional` or `@Cacheable`? How did you fix it?**
> Yes — calling an `@Transactional` method from another method in the *same* class bypasses the proxy entirely, since the call doesn't go through Spring's AOP interception. Fixes: restructure so the method lives in a separate bean and is called externally, inject a self-reference (`@Autowired` of the same type, or `ApplicationContext.getBean(...)`), or use `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)`. I generally prefer the "extract to another bean" approach — it's cleaner and signals intent better.

**Q11. How do you structure configuration across dev/staging/prod environments?**
> Spring profiles (`application-{profile}.yml`) for environment-specific overrides, externalized secrets via a vault (HashiCorp Vault, AWS Secrets Manager, or Spring Cloud Config Server backed by git), and `@ConfigurationProperties` classes with validation (`@Validated` + JSR-303 annotations) instead of scattering `@Value` everywhere — it's type-safe and fails at startup if something's misconfigured rather than failing at runtime when the property is actually used.

**Q12. What's your approach to API versioning in a Spring Boot service that's been live for years?**
> URI versioning (`/v1/`, `/v2/`) for major breaking changes — simplest for clients to understand and for routing/gateway rules. Header or content-negotiation versioning when changes are additive and you want to avoid URL churn. I avoid versioning everything at once; I version the resource that's actually breaking, and I keep old versions alive with a deprecation timeline communicated to consumers, with metrics on old-version traffic so I know when it's safe to sunset.

---

## 3. Microservices Architecture

**Q13. How do you decide service boundaries? What's gone wrong when boundaries were drawn badly?**
> I draw boundaries around business capabilities/domains (DDD bounded contexts), not technical layers — a service should own a cohesive piece of business data and logic end to end. The smell of a bad boundary: two services that always deploy together, or a "god" entity that five services all read/write directly. I've seen an "Order" service get split purely by CRUD operation across teams — that created chatty synchronous calls for every transaction and circular dependencies. We re-merged around the actual order lifecycle and pushed reporting concerns out to a separate read-optimized service via events instead.

**Q14. Synchronous (REST/gRPC) vs asynchronous (Kafka events) communication between services — how do you decide?**
> Synchronous when the caller needs an immediate answer to proceed (e.g., checking inventory before confirming an order) — but I keep these calls resilient with timeouts, circuit breakers, and bulkheads since a sync chain is only as available as its weakest link. Asynchronous/event-driven when the operation is a fact that happened and other services need to react eventually but not block the originator — e.g., "OrderPlaced" event triggering billing, notifications, and analytics independently. Async also decouples deployment and scaling — the order service doesn't care if the email service is down.

**Q15. How do you handle data consistency across services without distributed transactions (2PC)?**
> Saga pattern. Orchestration-based saga for complex multi-step flows where I want centralized visibility and explicit compensation logic — a saga orchestrator calls each service and issues compensating actions on failure (e.g., refund payment if shipping fails). Choreography-based saga for simpler flows where each service reacts to events and emits its own — less central coupling but harder to trace/debug as the chain grows. I also lean on the Outbox pattern to atomically persist a DB write and the corresponding event in the same local transaction, then a separate process (CDC like Debezium, or a polling publisher) reliably publishes it — this avoids the classic "DB commit succeeded but Kafka publish failed" dual-write problem.

**Q16. Explain idempotency — where does it matter and how have you implemented it?**
> It matters anywhere a retry can happen — client retries, Kafka consumer redelivery, network timeouts where the caller doesn't know if the first request succeeded. Implementation approaches: idempotency keys (client-generated UUID stored with the result, so a repeat request with the same key returns the cached result instead of reprocessing), natural idempotency in the operation itself (PUT-style "set to X" instead of "increment by X"), and a unique constraint at the DB level on a business key that throws and is treated as a no-op rather than a duplicate. For payment processing specifically, I always required an idempotency key on the API — non-negotiable.

**Q17. How do you handle service discovery and configuration in your microservices setup?**
> Depends on platform. In Kubernetes, I lean on native service discovery (DNS-based, via Services) rather than a separate registry like Eureka — fewer moving parts. For config, externalized config server (Spring Cloud Config or Kubernetes ConfigMaps/Secrets) so config changes don't require redeployment, with `@RefreshScope` for beans that need to pick up changes live, although in practice I'm cautious with hot-reload for things that affect connection pools or thread pools.

**Q18. What's your strategy for handling a cascading failure when one downstream microservice goes down?**
> Circuit breaker (Resilience4j) wrapping the call so after a failure threshold it opens and fails fast instead of piling up threads waiting on a dead dependency. Timeouts on every outbound call — no unbounded waits. Bulkheads (separate thread pools per dependency) so a slow dependency doesn't starve threads needed for healthy ones. Fallback responses where a degraded experience beats no response — e.g., serve cached/stale recommendation data if the recommendation service is down rather than failing the whole page. And at the platform level: proper readiness/liveness probes so Kubernetes pulls unhealthy instances out of rotation.

**Q19. How do you approach distributed tracing and debugging an issue that spans 6 microservices?**
> A correlation/trace ID generated at the edge (API gateway) and propagated through every hop — HTTP headers for sync calls, Kafka message headers for async. OpenTelemetry instrumentation with a backend like Jaeger or Zipkin to visualize the full trace as a waterfall, so I can see exactly where latency or failure occurred. Structured logging (JSON logs with the trace ID as a field) shipped to a centralized store (ELK/Loki) so I can pivot from "trace ID X failed" to the actual log lines across all 6 services in one query, instead of SSH-ing into six boxes.

**Q20. How do you handle schema/contract evolution between services without breaking consumers?**
> Consumer-driven contract testing (Pact) so producer changes are verified against actual consumer expectations in CI before merge. For event schemas specifically, a schema registry (Confluent Schema Registry with Avro/Protobuf) enforcing backward-compatible evolution rules — adding optional fields is fine, removing or renaming required fields breaks compatibility and fails the build. I treat any breaking schema change as requiring a new version/topic, never an in-place mutation.

---

## 4. Apache Kafka

**Q21. Explain how Kafka guarantees message ordering. What can break that guarantee?**
> Ordering is only guaranteed within a single partition, not across the topic. Producers send messages with the same key to the same partition (default partitioner hashes the key), so as long as related events share a key (e.g., all events for one order use orderId as key), they land in order in that partition and are consumed in order by one consumer. What breaks it: increasing partition count on an existing topic (changes the hash-to-partition mapping going forward, so old and new messages for the same key can land differently), retries with `max.in.flight.requests.per.connection > 1` without idempotent producer enabled (out-of-order retries), or multiple consumer instances in the same group processing the same partition concurrently via async processing without preserving order downstream.

**Q22. Walk me through consumer group rebalancing — what triggers it and what's the cost?**
> Triggers: a consumer joins/leaves the group, a consumer is considered dead (missed `session.timeout.ms` heartbeats), or partition count changes. Cost: with the older eager rebalancing protocol, *all* consumers in the group stop processing during the rebalance ("stop-the-world"), which can spike latency. Newer cooperative sticky rebalancing (`CooperativeStickyAssignor`) minimizes this — it only reassigns the partitions that actually need to move, letting unaffected consumers keep processing. I always set this assignor explicitly rather than relying on the default, and I tune `session.timeout.ms`/`max.poll.interval.ms` carefully — too aggressive and you get rebalance storms from slow processing being misread as dead consumers.

**Q23. How do you achieve exactly-once processing semantics with Kafka? Is it ever truly "exactly once"?**
> Kafka provides exactly-once *within* its own ecosystem via idempotent producers (`enable.idempotence=true`, deduped by producer ID + sequence number) and transactional producers (atomic writes across multiple partitions/topics, used heavily in Kafka Streams). For end-to-end exactly-once including an external system (a DB write triggered by a consumed message), true exactly-once is effectively impossible to guarantee without idempotent consumers — what you actually build is "effectively-once": at-least-once delivery + idempotent processing on the consumer side (dedupe by message key/offset stored in the same transaction as the business write). I'm honest about this distinction in design docs — claiming "exactly-once" without idempotent consumer logic is a false guarantee.

**Q24. Compare at-most-once, at-least-once, and exactly-once from a consumer commit perspective.**
> At-most-once: commit offset *before* processing — if processing fails, message is lost, never reprocessed. Rarely justified; only for non-critical telemetry-type data. At-least-once: process first, commit offset after — if the consumer crashes between processing and commit, the message gets redelivered and reprocessed, hence "at least" once. This is the default I use, paired with idempotent processing logic. Exactly-once (within Kafka): use a transactional consumer-producer pattern where the offset commit and the produced output are part of the same Kafka transaction — common in Kafka Streams or a Processor API setup.

**Q25. How do you size partitions for a topic? What happens if you under- or over-provision?**
> Partition count drives max parallelism — you can't have more active consumers in a group than partitions (extra consumers sit idle). I size based on target throughput ÷ per-partition throughput, with headroom for growth, since increasing partitions later breaks key-to-partition ordering guarantees for existing data. Under-provisioning caps your consumption throughput and creates consumer lag under load. Over-provisioning isn't free either — more partitions means more file handles and memory on brokers, longer leader election/rebalance times, and more open connections; I've seen clusters with thousands of needlessly small-traffic partitions struggle with broker overhead.

**Q26. A consumer is lagging badly in production — walk me through your diagnosis.**
> First, check consumer lag metrics (via `kafka-consumer-groups.sh` or a monitoring dashboard) to see if it's one partition or all — one partition lagging while others are fine usually points to a "poison pill" message or skewed key distribution sending too much traffic to one partition. Then check consumer-side: is processing per message slow (DB calls, downstream HTTP calls blocking the poll loop)? Check `max.poll.interval.ms` vs actual processing time — if processing exceeds it, the consumer gets kicked from the group and triggers a rebalance, making lag worse. Fixes I've used: scale out consumers (up to partition count), move slow synchronous work to async/batch processing, increase `max.poll.records` tuning alongside processing time, or split a hot key's traffic across more partitions with better key design.

**Q27. How do you handle a "poison pill" message that keeps crashing your consumer?**
> Without protection, the consumer crashes, retries from the same offset, crashes again — an infinite loop that halts that partition entirely. Pattern: wrap processing in a retry with backoff for a bounded number of attempts (transient errors recover), and on exhausting retries, route the message to a dead-letter topic (DLT) with error metadata (exception, original headers, retry count) instead of blocking the partition, then commit the offset and move on. Spring Kafka's `DefaultErrorHandler` with a `DeadLetterPublishingRecoverer` does exactly this declaratively. The DLT then gets monitored/alerted on, and I usually build a small reprocessing tool to replay DLT messages once the root cause is fixed.

**Q28. Kafka Streams vs a consumer + custom processing — when would you choose Kafka Streams?**
> Kafka Streams when the logic is inherently stream processing — stateful aggregations, windowed joins, or building a materialized view from an event stream (e.g., real-time running totals, joining order events with inventory events within a time window). It gives you fault-tolerant local state stores backed by changelog topics, exactly-once support, and a higher-level DSL (`KStream`, `KTable`) instead of hand-rolling state management. Plain consumer + custom code when the per-message logic is simple, stateless, or mostly about calling out to other services — Streams adds complexity (state stores, RocksDB, rebalancing of state) that isn't worth it for stateless transform-and-forward work.

**Q29. How do you ensure message delivery guarantees survive a broker failure?**
> Replication factor ≥ 3 in production, `min.insync.replicas=2`, and producer `acks=all` — the producer only considers a write successful once it's acknowledged by all in-sync replicas, not just the leader, so a single broker failure right after a write doesn't lose data. On the consumer side, combined with at-least-once processing and idempotent handling, the system tolerates broker failure without silent data loss — though it does add latency vs `acks=1`, which is the trade-off I explicitly call out to stakeholders when they ask "why is this slower than it could be."

**Q30. How would you design a topic strategy for a multi-tenant system?**
> Depends on tenant count and isolation needs. Shared topic with tenant ID as part of the message key/payload works well for moderate tenant counts and lets you scale partitions independently of tenant count — but requires careful access control at the application layer since Kafka ACLs are topic-level, not message-level. Per-tenant topics give hard isolation (separate retention, ACLs, throughput limits per tenant) but doesn't scale past a few hundred tenants before topic/partition sprawl becomes a cluster management problem. For a B2B platform with a manageable number of large tenants needing strict isolation, I'd lean per-tenant topics; for a high-volume consumer-facing system with thousands of tenants, shared topics with strong key-based partitioning.

---

## 5. Databases, Transactions & Data Patterns

**Q31. How do you decide between SQL and NoSQL for a new microservice?**
> Start from access patterns, not preference. Strong relational integrity, complex joins/queries, ACID transactions across entities → relational (Postgres/MySQL). High write throughput, flexible/evolving schema, simple key-based access patterns, or needing to scale horizontally beyond what a single relational instance handles → NoSQL (Mongo for document flexibility, Cassandra/DynamoDB for very high write throughput and predictable key-based access). In a microservices architecture I usually end up with a polyglot mix — the order service on Postgres for transactional integrity, a read-optimized search/catalog service on Elasticsearch or a denormalized NoSQL store fed by events from the source of truth.

**Q32. Explain database-per-service. How do you handle reporting/analytics queries that used to be a simple JOIN?**
> Each service owns its own schema/database, no other service touches it directly — that's the whole point of bounded contexts and independent deployability. For cross-service queries that used to be a SQL JOIN: CQRS with a read-model — a separate service/store subscribes to domain events from all the relevant services and builds a denormalized view purpose-built for the query (e.g., an "order summary with customer and product details" materialized view updated via Kafka events). For ad hoc analytics, a data warehouse fed by CDC (Debezium) streaming each service's changes, so analysts query the warehouse, not production databases.

**Q33. What's your approach to database migrations in a live microservices system with zero downtime?**
> Tools like Flyway/Liquibase versioned in source control, run as part of deploy pipeline. For zero-downtime schema changes: expand-contract pattern — add new column/table without removing old (expand), deploy code that writes to both old and new, backfill data, deploy code that reads from new exclusively, then later remove the old column (contract) in a separate deploy. Never a single migration that both adds and drops in a way that breaks the currently-running old version of the app during a rolling deploy.

**Q34. How have you handled the N+1 query problem in a Spring Data JPA service?**
> Diagnosed via Hibernate SQL logging or APM showing repeated near-identical queries in a loop. Fixes depend on context: `JOIN FETCH` in JPQL for a known fixed relationship needed eagerly, `@EntityGraph` for more declarative fetch control without changing the default fetch type globally, or batch fetching (`hibernate.default_batch_fetch_size`) when you can't avoid lazy loading but want it batched instead of one-by-one. I avoid blanket `FetchType.EAGER` on entity mappings — it just moves the N+1 problem earlier and applies it everywhere that entity is loaded, even where you didn't need the association.

---

## 6. System Design / Scenario-Based

**Q35. Design a system that processes order events and updates inventory, with the requirement that inventory never goes negative under concurrent load.**
> I'd walk through: order service publishes `OrderPlaced` to Kafka, keyed by productId so all events for a product hit the same partition (ordering guarantee). Inventory service consumes and decrements within a DB transaction using either optimistic locking (version column, retry on conflict) or a conditional update (`UPDATE inventory SET qty = qty - ? WHERE qty >= ?`, checking rows affected) to prevent negative stock under concurrent decrements — pessimistic row locks if contention is very high on hot SKUs. If stock is insufficient, publish `InventoryReservationFailed`, triggering the saga's compensating action back on the order. I'd also flag the trade-off here: keying by productId concentrates a hot SKU's traffic on one partition — worth discussing capacity for flash-sale-type spikes.

**Q36. How would you design rate limiting across a fleet of microservices behind an API gateway?**
> Centralized at the gateway (Kong, Spring Cloud Gateway, or a cloud API gateway) using a token bucket or sliding window algorithm backed by Redis so limits are consistent across multiple gateway instances, not per-instance in-memory counters which would let traffic multiply by instance count. Per-client/API-key limits plus a global circuit-breaker-style limit to protect backend services from aggregate overload regardless of per-client fairness. For internal service-to-service calls I'd add a secondary layer with Resilience4j rate limiters as defense in depth, since the gateway protects the edge but not service-to-service chatter.

**Q37. How do you approach capacity planning and load testing for a new high-traffic service before launch?**
> Define SLOs first (p99 latency target, target RPS, error budget) from product requirements, not guesses. Load test with realistic traffic shape (Gatling/JMeter/k6) including burst patterns, not just steady-state average load. Identify the actual bottleneck under load — usually not the app tier itself but a downstream dependency (DB connection pool exhaustion, a synchronous third-party call, Kafka producer buffer). I size thread pools, connection pools, and JVM heap based on observed behavior under that load test, not theoretical math, then validate again after tuning. I always test failure modes too — what happens when a dependency is slow, not just when it's healthy.

---

## 7. Behavioral / Leadership (Senior-Level Signal)

**Q38. Tell me about a time you disagreed with an architectural decision your team or lead made.**
> Structure your real answer with STAR, but the substance interviewers want at senior level: did you bring data/trade-offs instead of opinion, did you escalate respectfully, did you commit once the decision was made even if it wasn't yours, and what was the actual outcome. Avoid a story where you were simply "right" and everyone agreed — that's not a disagreement, it's a compliance story. A strong answer: you pushed back on async-everywhere because it added debugging complexity for a low-traffic internal service, presented the operational cost, lost the argument, but then made sure tracing/observability was in place to mitigate the downside you'd flagged.

**Q39. Describe the most significant production incident you've handled. What did you change afterward?**
> Senior signal here is less about the heroics of the fix and more about: how fast you correctly diagnosed root cause vs symptom, how you communicated during the incident (status updates, not silence), and what systemic change came out of the postmortem — not "we fixed the bug" but "we added a circuit breaker / changed our deploy process / added an alert that would have caught this earlier." Blameless postmortem culture is a strong thing to mention if it's true for you.

**Q40. How do you mentor or bring up junior/mid-level engineers on your team?**
> Concrete examples interviewers look for: pairing on design before code is written rather than only reviewing PRs after the fact, giving juniors ownership of a real (bounded) decision rather than just tickets, code review that explains *why* not just *what*, and creating space for them to make a recoverable mistake rather than you doing it for them. Avoid vague "I just help them when they're stuck" — give one specific story with a measurable outcome (e.g., someone you mentored went on to own a service independently).

---

## How to Use This for Final Prep

- Have **2–3 real production stories** ready (an incident, a migration, a design decision) you can adapt to multiple behavioral questions — don't memorize answers verbatim, know the shape of the story.
- For every Kafka/microservices answer, be ready for the immediate follow-up: *"What would you do differently if you rebuilt it today?"* — seniors are expected to self-critique their own past decisions.
- Practice drawing your saga/event-flow answers on a whiteboard or shared doc — system design rounds for this level are usually live diagramming, not pure Q&A.
