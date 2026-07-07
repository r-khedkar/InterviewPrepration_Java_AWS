# Interview Preparation

### Senior Java Backend Engineer --- Mastercard

**Contents:** System Design · Spring Boot Deep Dives · Coding · Behavioral · Caching In Depth

# Section 1: Mastercard --- Senior Java Backend Engineer Interview Prep

### System Design · Spring Boot Deep Dives · Coding · Behavioral

## PART 1 --- System Design & Architecture

### Q1. How would you design a payment processing system for high scalability?

**Core principles, not just components:**

1.  **Idempotency is non-negotiable.** Every payment request carries a client-generated idempotency key. Network retries, double-clicks, or gateway timeouts must never result in double-charging. Store the key + result in a fast lookup (Redis or a DB unique constraint) and short-circuit repeats.

2.  **Asynchronous processing for anything non-blocking.** The synchronous path is: validate request → reserve/authorize (this part *must* be sync, the user needs an answer) → return "authorized" immediately. Everything after --- settlement, ledger updates, notifications, fraud scoring refinement, reconciliation --- happens async via Kafka events, decoupled from the user-facing latency.

3.  **Saga pattern for the multi-step flow** (authorize → capture → settle → reconcile), with explicit compensating actions (reversal/refund) at each step if a downstream step fails. Never a 2-phase-commit across services --- too slow and fragile at this scale.

4.  **Outbox pattern** so the DB write (payment record) and the event publish (to Kafka) are atomic --- avoiding the classic "DB committed, Kafka publish failed" gap that would silently lose an event.

5.  **Partition by a high-cardinality key** (e.g., merchantId or accountId) for both DB sharding and Kafka topic partitioning, so load spreads evenly and you don't get hot shards from one massive merchant.

6.  **Strong consistency where money moves, eventual consistency everywhere else.** The ledger/balance update needs ACID guarantees (a relational DB with row-level locking or optimistic concurrency on the account row). Things like "show transaction history in the app" can be eventually consistent, fed by CDC into a read store.

7.  **Defense in depth for resiliency:** circuit breakers + timeouts on every external call (card networks, banks), bulkheads so a slow downstream bank doesn't starve threads needed for healthy ones, and a dead-letter topic for events that fail processing after retries --- never silently drop a payment event.

8.  **Reconciliation as a first-class process**, not an afterthought --- a scheduled job that compares your internal ledger against the source-of-truth statements from card networks/banks and flags discrepancies, because at payment scale, *something* will eventually disagree.

**The trade-off to call out explicitly:** the system favors strong consistency on the money-moving path and accepts complexity (sagas, compensations) in exchange for not needing distributed transactions --- that's the right trade at scale, but it means you must design every step to be safely retryable and reversible from day one.

### Q2. Explain different transaction propagation levels in Spring Boot.

Propagation defines how a `@Transactional` method behaves when called from another method that's *already* in a transaction.

  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Propagation                         Behavior
  ----------------------------------- ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  `REQUIRED` (default)                Joins the existing transaction if one exists; creates a new one if not. Most common --- one logical unit of work.

  `REQUIRES_NEW`                      Always suspends any existing transaction and starts a brand new, independent one. Use when a sub-operation must commit/rollback independently --- e.g., writing an audit log that should persist even if the parent transaction later rolls back.

  `NESTED`                            Creates a savepoint within the existing transaction. If the nested part fails, it rolls back to the savepoint without killing the outer transaction (only works with JDBC, not all JPA providers support it well).

  `SUPPORTS`                          Joins existing transaction if present, runs non-transactionally if not. Rarely used deliberately.

  `NOT_SUPPORTED`                     Suspends any existing transaction, runs the method without one. Useful for read-only reporting calls that shouldn't hold a lock.

  `MANDATORY`                         Must run within an existing transaction --- throws an exception if called outside one. Used to enforce "this method should never be called standalone."

  `NEVER`                             Must run *without* a transaction --- throws if one exists. Rare; used for operations that must never be wrapped (e.g., calls that manage their own commit behavior).
  ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    @Service
    public class PaymentService {

        @Transactional(propagation = Propagation.REQUIRED)
        public void processPayment(Payment payment) {
            ledgerRepository.save(payment);       // part of this transaction
            auditService.logPaymentAttempt(payment); // separate transaction — see below
        }
    }

    @Service
    public class AuditService {

        // REQUIRES_NEW: audit log commits independently —
        // even if processPayment() later rolls back, the audit trail survives
        @Transactional(propagation = Propagation.REQUIRES_NEW)
        public void logPaymentAttempt(Payment payment) {
            auditRepository.save(new AuditEntry(payment));
        }
    }

### Q3. How does `@ControllerAdvice` help in exception handling?

It's a global, centralized exception handler --- instead of writing try/catch in every controller method, you define handler methods once and they apply across *all* (or selected) controllers.

    @RestControllerAdvice
    public class GlobalExceptionHandler {

        @ExceptionHandler(PaymentNotFoundException.class)
        public ResponseEntity<ErrorResponse> handleNotFound(PaymentNotFoundException ex) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("PAYMENT_NOT_FOUND", ex.getMessage()));
        }

        @ExceptionHandler(InsufficientFundsException.class)
        public ResponseEntity<ErrorResponse> handleInsufficientFunds(InsufficientFundsException ex) {
            return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED)
                .body(new ErrorResponse("INSUFFICIENT_FUNDS", ex.getMessage()));
        }

        @ExceptionHandler(Exception.class)   // catch-all fallback
        public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
        }
    }

**Why it matters at scale:** consistent error response shape across every endpoint (important for API consumers), no duplicated error-handling boilerplate in 50 controllers, and you can scope it (`@ControllerAdvice(basePackages = "...")`) if different modules need different handling.

### Q4. What are the different isolation levels in Spring transactions, and when would you use each?

Isolation controls how much one transaction can see of another transaction's *uncommitted* changes --- Spring just delegates to the underlying DB's isolation levels.

  --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Level                Prevents                            Allows                                             When to use
  -------------------- ----------------------------------- -------------------------------------------------- --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  `READ_UNCOMMITTED`   Nothing                             Dirty reads, non-repeatable reads, phantom reads   Almost never in production --- reading uncommitted/possibly-rolled-back data is dangerous. Maybe for rough analytics where perfect accuracy doesn't matter.

  `READ_COMMITTED`     Dirty reads                         Non-repeatable reads, phantom reads                The most common default (Postgres's default). Good balance --- you never see uncommitted data, but two reads in the same transaction might differ if another transaction commits in between.

  `REPEATABLE_READ`    Dirty reads, non-repeatable reads   Phantom reads                                      When you need the same query to return the same rows throughout a transaction --- e.g., reading an account balance multiple times during a transfer and needing it to stay consistent. MySQL's default.

  `SERIALIZABLE`       All of the above                    Nothing                                            Strongest, slowest. Transactions execute as if run one at a time. Use for genuinely critical financial operations where any anomaly is unacceptable --- at the cost of throughput and higher chance of deadlocks/retries under contention.
  --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferFunds(String fromAccount, String toAccount, BigDecimal amount) {
        // Critical money-movement logic — willing to pay the throughput cost
        // for the strongest consistency guarantee here.
    }

**Senior-level point to make:** isolation level is a trade-off knob, not a "more is always better" setting --- `SERIALIZABLE` everywhere will tank your throughput. I pick the lowest isolation level that still prevents the specific anomaly that matters for that operation, and use optimistic locking (`@Version`) instead of pessimistic isolation where contention is low.

### Q5. Explain the concept of idempotency in APIs.

An idempotent operation produces the same result no matter how many times it's executed --- calling it once or calling it five times (due to retries, network blips, double-clicks) leaves the system in the same state.

    @RestController
    public class PaymentController {

        private final IdempotencyKeyRepository idempotencyRepo;
        private final PaymentService paymentService;

        @PostMapping("/payments")
        public ResponseEntity<PaymentResponse> createPayment(
                @RequestHeader("Idempotency-Key") String idempotencyKey,
                @RequestBody PaymentRequest request) {

            // Check if we've already processed this exact request
            Optional<PaymentResponse> existing = idempotencyRepo.findResponseByKey(idempotencyKey);
            if (existing.isPresent()) {
                return ResponseEntity.ok(existing.get()); // return cached result, don't reprocess
            }

            PaymentResponse response = paymentService.process(request);
            idempotencyRepo.save(idempotencyKey, response); // store before returning
            return ResponseEntity.ok(response);
        }
    }

**Natural idempotency vs key-based idempotency:** - `PUT /accounts/123 { "status": "active" }` is naturally idempotent --- setting to a fixed state, repeating it changes nothing further. - `POST /payments` (charge \$50) is **not** naturally idempotent --- that's exactly why it needs an explicit idempotency key, since "charge \$50" repeated means "charge \$50 again."

### Q6. What is the difference between `HashMap` and `ConcurrentHashMap`?

  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
                                  `HashMap`                                                                                 `ConcurrentHashMap`
  ------------------------------- ----------------------------------------------------------------------------------------- --------------------------------------------------------------------------------------------------------------------------------------------
  Thread safety                   None --- concurrent modification can corrupt internal structure or cause infinite loops   Thread-safe without locking the entire map

  Null keys/values                Allows one null key, multiple null values                                                 Disallows null keys and null values entirely (ambiguous in concurrent context --- can't tell "not present" from "present but null")

  Locking                         N/A                                                                                       Internally uses fine-grained locking/CAS on bins, not a single global lock --- multiple threads can write to different bins simultaneously

  Iteration                       `ConcurrentModificationException` if modified during iteration                            Weakly consistent iterator --- never throws, may or may not reflect concurrent updates made during iteration

  Performance (single-threaded)   Faster --- no synchronization overhead                                                    Slightly slower due to thread-safety machinery
  ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    // HashMap in a multithreaded context — UNSAFE
    Map<String, Integer> unsafe = new HashMap<>();
    // Two threads calling unsafe.put() concurrently can corrupt the bucket structure

    // ConcurrentHashMap — safe for concurrent reads/writes
    Map<String, Integer> safe = new ConcurrentHashMap<>();
    safe.computeIfAbsent("count", k -> 0);
    safe.compute("count", (k, v) -> v + 1);   // atomic read-modify-write, no external lock needed

**When I'd still wrap a** `HashMap` **in** `Collections.synchronizedMap`**:** almost never anymore --- `ConcurrentHashMap` is strictly better for typical use cases. The only reason to reach for `synchronizedMap` is needing to synchronize multiple operations together as one atomic block (e.g., "check size, then add if under limit" needs an external lock regardless of map type).

### Q7. What is the difference between `Callable` and `Runnable`?

    // Runnable: no return value, cannot throw checked exceptions
    Runnable task1 = () -> {
        System.out.println("Running, no result, no checked exceptions allowed");
    };

    // Callable: returns a value, CAN throw checked exceptions
    Callable<Integer> task2 = () -> {
        if (someCondition) throw new IOException("simulated failure"); // allowed!
        return 42;
    };

    ExecutorService executor = Executors.newFixedThreadPool(2);

    executor.submit(task1);                       // returns Future<?>, result always null
    Future<Integer> future = executor.submit(task2); // returns Future<Integer> with actual result

    try {
        Integer result = future.get();   // blocks until done, can throw ExecutionException
    } catch (ExecutionException e) {
        Throwable cause = e.getCause();  // the original IOException is wrapped here
    }

**Key differences:** - `Runnable.run()` returns `void`; `Callable.call()` returns `V` (generic). - `Runnable` can only throw unchecked exceptions; `Callable` can throw checked exceptions, which get wrapped in `ExecutionException` when retrieved via `future.get()`. - Both can be submitted to an `ExecutorService`, but only `Callable` gives you a meaningful `Future<V>` result.

### Q8. How does garbage collection work in Java?

**The core idea:** Java tracks object reachability from GC roots (active thread stacks, static references, JNI references). Anything unreachable is garbage and gets reclaimed --- you never manually `free()`.

**Generational hypothesis** (most collectors are built on this): most objects die young. So the heap is split: - **Young Generation** (Eden + two Survivor spaces) --- new objects allocated here. Minor GC runs frequently, is fast, and copies surviving objects between survivor spaces, promoting long-lived ones to Old Gen. - **Old Generation (Tenured)** --- long-lived objects. Major/Full GC runs less often but takes longer since it scans a much larger space.

**Modern collectors (you should know the trade-offs, not just names):** - **G1 (default since Java 9)** --- divides the heap into regions, aims for low pause times with a target max pause goal you can configure (`-XX:MaxGCPauseMillis`). Good general-purpose choice. - **ZGC / Shenandoah** --- sub-millisecond pause collectors designed for very large heaps and latency-sensitive applications --- the trade-off is somewhat higher CPU overhead for the concurrent work. - **Parallel GC** --- optimizes for throughput over pause time, uses multiple threads but stops the world during collection --- fine for batch jobs where pause time doesn't matter.

    # Choosing a collector and tuning pause goals (JVM flags, not code)
    java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar app.jar

**Senior-level point:** GC tuning is a symptom-treatment, not a cure --- if you're constantly fighting GC pauses, the real fix is usually reducing allocation rate (object pooling, avoiding unnecessary boxing, streaming large datasets instead of loading them fully into memory) rather than just picking a "better" collector.

### Q9. Explain the lifecycle of a Spring Bean.

1.  **Instantiation** --- Spring creates the object (via constructor).
2.  **Populate properties** --- dependencies injected (constructor injection happens during instantiation; field/setter injection happens here).
3.  `BeanNameAware`**/**`BeanFactoryAware`**/**`ApplicationContextAware` callbacks if the bean implements them (rare, mostly framework-internal use).
4.  `@PostConstruct` method invoked (or `InitializingBean.afterPropertiesSet()`) --- your hook for post-injection setup logic.
5.  **Bean is ready** --- fully initialized, available for use, registered in the application context.
6.  `@PreDestroy` method invoked (or `DisposableBean.destroy()`) when the context shuts down --- your hook for cleanup (closing connections, flushing buffers).

```{=html}
<!-- -->
```
    @Component
    public class PaymentGatewayClient {

        private HttpClient httpClient;

        public PaymentGatewayClient(GatewayConfig config) {
            // constructor injection — dependencies set here
        }

        @PostConstruct
        public void init() {
            // called AFTER dependency injection is complete, BEFORE bean is used anywhere
            this.httpClient = HttpClient.newBuilder().connectTimeout(...).build();
        }

        @PreDestroy
        public void cleanup() {
            // called when the application context is shutting down
            httpClient = null; // or close any pooled connections
        }
    }

**Beginner gotcha:** don't put initialization logic that depends on injected fields directly in the constructor if those fields come from setter/field injection --- they won't be set yet. `@PostConstruct` is the safe place for "now that everything's wired, do this."

### Q10. What is Dependency Injection, and why is it important in Spring?

**The problem it solves:** without DI, a class creates its own dependencies directly (`new PaymentGateway()` inside the service) --- tightly coupling the class to a specific implementation, making it hard to swap implementations or test in isolation (you can't mock a `new`-ed dependency easily).

**With DI**, the class declares what it needs, and a container (Spring) provides it.

    // WITHOUT DI — tightly coupled, hard to test
    public class PaymentService {
        private StripeGateway gateway = new StripeGateway(); // hardcoded, can't swap or mock
    }

    // WITH DI — loosely coupled, container provides the dependency
    @Service
    public class PaymentService {
        private final PaymentGateway gateway;   // interface, not concrete class

        public PaymentService(PaymentGateway gateway) {  // Spring injects whichever impl is configured
            this.gateway = gateway;
        }
    }

    @Component
    public class StripeGateway implements PaymentGateway { /* ... */ }

**Why it matters in practice:** unit tests become trivial (`new PaymentService(mockGateway)` --- no Spring context needed), swapping implementations (Stripe → internal gateway) means changing a bean definition, not the consuming code, and it enforces the Dependency Inversion Principle (depend on abstractions, not concrete classes).

### Q11. How does the Spring `@Transactional` annotation work?

Under the hood, `@Transactional` is implemented via **AOP proxies**. When a bean has `@Transactional` methods, Spring wraps it in a proxy (CGLIB subclass proxy by default for classes, or a JDK dynamic proxy if the bean implements an interface). The proxy intercepts the method call, starts a transaction (begins a DB transaction via the configured `PlatformTransactionManager`), invokes the real method, then commits on success or rolls back on a qualifying exception.

    @Service
    public class PaymentService {

        @Transactional
        public void processPayment(Payment payment) {
            // What actually happens:
            // 1. Proxy intercepts this call
            // 2. transactionManager.getTransaction() — begins transaction
            // 3. Your actual method body runs
            // 4a. No exception → transactionManager.commit()
            // 4b. RuntimeException thrown → transactionManager.rollback()
            ledgerRepository.save(payment);
            accountRepository.debit(payment.getAccountId(), payment.getAmount());
        }
    }

**The two gotchas every senior candidate should know cold:** 1. **Self-invocation bypasses the proxy** --- calling `this.processPayment()` from another method *in the same class* skips the proxy entirely, so no transaction is started. 2. **Only unchecked exceptions trigger rollback by default** --- a checked exception commits the transaction unless you specify `@Transactional(rollbackFor = Exception.class)`.

### Q12. Java Streams --- multiple hands-on patterns

    List<Order> orders = List.of(
        new Order("O1", "alice", 250.0, "COMPLETED"),
        new Order("O2", "bob", 120.0, "PENDING"),
        new Order("O3", "alice", 90.0, "COMPLETED"),
        new Order("O4", "carol", 500.0, "COMPLETED"),
        new Order("O5", "bob", 75.0, "COMPLETED")
    );

    // 1. Filter + map + collect: get all completed order amounts
    List<Double> completedAmounts = orders.stream()
        .filter(o -> o.status().equals("COMPLETED"))
        .map(Order::amount)
        .collect(Collectors.toList());

    // 2. Group by customer, sum their order totals
    Map<String, Double> totalByCustomer = orders.stream()
        .filter(o -> o.status().equals("COMPLETED"))
        .collect(Collectors.groupingBy(Order::customer, Collectors.summingDouble(Order::amount)));
    // {alice=340.0, bob=75.0, carol=500.0}

    // 3. Find the highest-value order per customer
    Map<String, Optional<Order>> topOrderByCustomer = orders.stream()
        .collect(Collectors.groupingBy(Order::customer,
            Collectors.maxBy(Comparator.comparingDouble(Order::amount))));

    // 4. Sort by amount descending, get top 3
    List<Order> top3 = orders.stream()
        .sorted(Comparator.comparingDouble(Order::amount).reversed())
        .limit(3)
        .collect(Collectors.toList());

    // 5. Chained: filter -> group -> count
    Map<String, Long> completedCountByCustomer = orders.stream()
        .filter(o -> o.status().equals("COMPLETED"))
        .collect(Collectors.groupingBy(Order::customer, Collectors.counting()));

    // 6. Partition into two groups (completed vs not) in one pass
    Map<Boolean, List<Order>> partitioned = orders.stream()
        .collect(Collectors.partitioningBy(o -> o.status().equals("COMPLETED")));

    // 7. Reduce: total revenue across all completed orders
    double totalRevenue = orders.stream()
        .filter(o -> o.status().equals("COMPLETED"))
        .map(Order::amount)
        .reduce(0.0, Double::sum);

    // 8. flatMap: flatten a list of lists (e.g., orders per customer -> all order items)
    List<List<String>> itemsPerOrder = List.of(List.of("A","B"), List.of("C"), List.of("D","E"));
    List<String> allItems = itemsPerOrder.stream()
        .flatMap(List::stream)
        .collect(Collectors.toList());
    // [A, B, C, D, E]

    // 9. joining: build a comma-separated summary string
    String summary = orders.stream()
        .map(Order::customer)
        .distinct()
        .collect(Collectors.joining(", ", "Customers: ", "."));
    // "Customers: alice, bob, carol."

    record Order(String id, String customer, double amount, String status) {}

**Senior-level point:** streams should read like a pipeline of intent (filter → transform → collect), not nested loops translated 1:1 into stream calls. If a stream chain needs a comment to explain what it's doing, it's usually a sign to break it into named intermediate variables or extract a method.

### Q13. What is the difference between `@RestController` and `@Controller`?

    @Controller
    public class WebController {
        @GetMapping("/home")
        public String home() {
            return "home";  // resolved as a VIEW NAME (e.g., home.html via Thymeleaf)
        }

        @GetMapping("/api/data")
        @ResponseBody   // must add this explicitly to return raw data instead of a view name
        public String data() {
            return "raw data";
        }
    }

    @RestController  // = @Controller + @ResponseBody on every method, automatically
    public class ApiController {
        @GetMapping("/api/payments/{id}")
        public PaymentResponse getPayment(@PathVariable String id) {
            return paymentService.find(id);  // automatically serialized to JSON, no view resolution
        }
    }

`@RestController` is a convenience meta-annotation --- `@Controller` + `@ResponseBody` combined, so every method's return value is written directly to the response body (typically as JSON) instead of being resolved as a view name. Use `@Controller` for server-rendered pages (Thymeleaf/JSP), `@RestController` for APIs --- which is virtually everything in a microservices architecture.

### Q14. How does Spring Boot handle security by default?

Just adding `spring-boot-starter-security` to your dependencies gives you, with **zero configuration**: - Every endpoint requires authentication. - A default in-memory user (`user`) with a randomly generated password printed to the console at startup. - A default login form for browser-based auth, and HTTP Basic auth support for non-browser clients. - CSRF protection enabled by default for state-changing requests. - Sensible default security headers (`X-Content-Type-Options`, `X-Frame-Options`, etc.) on every response. - Sessions managed securely (session fixation protection).

    // Overriding the defaults with your own configuration
    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .anyRequest().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults())); // JWT-based auth for APIs
            return http.build();
        }
    }

**Senior-level point:** the "secure by default" behavior is exactly why teams sometimes get confused in their first week with Spring Security --- they add the dependency, every endpoint suddenly returns 401, and they haven't yet written a `SecurityFilterChain` bean to define their actual rules.

### Q15. What is circuit breaker design in Microservices, and how would you implement it?

**The problem:** if Service A calls Service B, and B is slow/down, A's threads pile up waiting on B, eventually exhausting A's own thread pool --- A goes down too, even though A itself is healthy. This is cascading failure.

**The pattern:** track failures on calls to B. Once failures cross a threshold, "open" the circuit --- stop calling B entirely for a cooldown period, fail fast instead (or return a fallback). After the cooldown, allow a few "trial" calls through (half-open state) --- if they succeed, close the circuit and resume normal calls; if they fail, reopen it.

    @Service
    public class InventoryClient {

        @CircuitBreaker(name = "inventoryService", fallbackMethod = "fallbackInventory")
        @TimeLimiter(name = "inventoryService")
        public CompletableFuture<InventoryStatus> checkStock(String productId) {
            return CompletableFuture.supplyAsync(() ->
                restTemplate.getForObject("/inventory/" + productId, InventoryStatus.class)
            );
        }

        public CompletableFuture<InventoryStatus> fallbackInventory(String productId, Throwable t) {
            // Called when circuit is OPEN or the call failed/timed out
            return CompletableFuture.completedFuture(InventoryStatus.unknown(productId));
        }
    }
    # application.yml — Resilience4j config
    resilience4j:
      circuitbreaker:
        instances:
          inventoryService:
            failure-rate-threshold: 50          # open circuit if 50% of calls fail
            wait-duration-in-open-state: 10s    # how long to stay open before testing again
            sliding-window-size: 20             # evaluate failure rate over the last 20 calls
            permitted-number-of-calls-in-half-open-state: 5

### Q16. How would you optimize a query in a database with millions of rows?

1.  **Check** `EXPLAIN ANALYZE` **first** --- never guess. See if the query is doing a full table scan when it should be using an index.
2.  **Index the columns actually used in** `WHERE`**,** `JOIN`**, and** `ORDER BY` --- but be deliberate, since every index also slows down writes and uses disk/memory.
3.  **Composite indexes matching query patterns** --- if you always filter by `status` then `created_at`, a composite index `(status, created_at)` serves that better than two separate single-column indexes.
4.  **Avoid** `SELECT *` --- pull only needed columns, especially relevant if some columns are large (TEXT/BLOB) and unused in this query.
5.  **Pagination via keyset/cursor instead of** `OFFSET` --- `OFFSET 1000000` still has to scan and discard a million rows; a cursor (`WHERE id > :lastSeenId ORDER BY id LIMIT 50`) jumps straight there using the index.
6.  **Partition large tables** (by date range, tenant, etc.) so queries that naturally filter on the partition key only scan relevant partitions.
7.  **Denormalize/precompute for read-heavy access patterns** that don't need real-time accuracy --- a materialized view or a read-optimized table updated via CDC/events instead of computing aggregates live every time.
8.  **Watch for N+1 queries** at the application/ORM level --- one slow query is often easier to fix than 500 fast ones executed in a loop.

```{=html}
<!-- -->
```
    -- Before: full scan, slow at scale
    SELECT * FROM transactions WHERE merchant_id = 'M123' ORDER BY created_at DESC OFFSET 100000 LIMIT 50;

    -- After: composite index + keyset pagination
    CREATE INDEX idx_merchant_created ON transactions(merchant_id, created_at DESC);

    SELECT id, amount, created_at FROM transactions
    WHERE merchant_id = 'M123' AND created_at < :lastSeenTimestamp
    ORDER BY created_at DESC LIMIT 50;

### Q17. What are design patterns? Explain with an example.

Design patterns are proven, reusable solutions to recurring software design problems --- a shared vocabulary so "use a strategy pattern here" communicates a whole design instantly instead of re-explaining it.

**Strategy Pattern example** --- relevant to payments: different payment methods need different processing logic, selected at runtime.

    public interface PaymentStrategy {
        PaymentResult process(PaymentRequest request);
    }

    @Component("creditCard")
    public class CreditCardStrategy implements PaymentStrategy {
        public PaymentResult process(PaymentRequest request) {
            // credit-card-specific authorization logic
            return new PaymentResult("SUCCESS", "Charged via card network");
        }
    }

    @Component("bankTransfer")
    public class BankTransferStrategy implements PaymentStrategy {
        public PaymentResult process(PaymentRequest request) {
            // ACH/bank-transfer-specific logic
            return new PaymentResult("PENDING", "Bank transfer initiated, settles in 2-3 days");
        }
    }

    @Service
    public class PaymentProcessor {
        private final Map<String, PaymentStrategy> strategies; // Spring injects all beans keyed by name

        public PaymentProcessor(Map<String, PaymentStrategy> strategies) {
            this.strategies = strategies;
        }

        public PaymentResult process(String method, PaymentRequest request) {
            PaymentStrategy strategy = strategies.get(method);
            if (strategy == null) throw new UnsupportedPaymentMethodException(method);
            return strategy.process(request);
        }
    }

**Other patterns worth having ready:** Factory (creating objects without exposing instantiation logic --- e.g., a `NotificationFactory` returning email/SMS/push senders), Builder (constructing complex immutable objects step by step --- common for request/config objects with many optional fields), Observer (Spring's `ApplicationEventPublisher`/`@EventListener` is this pattern built in), Singleton (Spring beans are singleton-scoped by default).

## PART 2 --- Behavioral & Mastercard-Specific

### Q18. Describe a challenging project you worked on and how you resolved any issues.

> Use STAR. Pick a story with real technical depth --- ideally something involving scale, a production incident, or a hard trade-off (consistency vs availability, a tight deadline vs technical debt). Senior interviewers want to hear your *reasoning process*, not just the fix --- what options you considered, why you picked one, what you'd do differently now. End with a measurable outcome (latency reduced X%, incident frequency dropped, etc.) if you have one.

### Q19. How do you prioritize tasks when working on multiple projects?

> Frame around impact and risk, not just deadlines: what's blocking other people (unblock first), what has the highest business/customer impact, what has a hard external deadline (regulatory, contractual) versus a soft internal one. Mention communicating trade-offs upward rather than silently absorbing overload --- a senior engineer flags "I can do A and B well by Friday, or A, B, and C poorly" rather than just trying to do everything.

### Q20. How would you handle a situation where your team is facing continuous production issues?

> Structure: stop the bleeding first (mitigate/rollback/feature-flag off, don't debug root cause live in prod under pressure), stabilize, then do a proper blameless postmortem to find the *systemic* cause, not just the immediate bug. If it's "continuous" (plural incidents), that's itself a signal --- usually points to a gap in testing, monitoring/alerting, or a fragile area of the architecture that needs investment, not just more firefighting. Mention pushing for time to address the systemic issue even if it competes with feature work --- that's often the actual senior-level point being tested.

### Q21. Why do you want to join Mastercard?

> Be specific and genuine rather than generic. Good angles: the scale and reliability bar of global payment infrastructure is a different category of engineering challenge than most companies offer (billions of transactions, strict latency/availability SLAs, real financial consequences for bugs), Mastercard's documented investment in cloud-native modernization and open banking/API initiatives, and if relevant to you personally --- the mission of expanding financial inclusion. Avoid generic "great brand, great opportunity" language; an interviewer can tell a templated answer from one with real specifics.

### Q22. What do you know about Mastercard's technological initiatives?

> Worth a quick search before the interview since this changes --- but areas historically associated with Mastercard's tech strategy: their open banking/API ecosystem investments (acquisitions like Finicity), tokenization technology for secure digital payments, fraud detection using AI/ML at massive transaction scale, and cloud infrastructure modernization. Tie it back to your own stack --- e.g., "the fraud-detection-at-scale problem is exactly the kind of high-throughput, low-latency stream processing I've worked on with Kafka" --- connects their initiatives to your actual experience instead of reciting facts.

### Q23. How would you optimize the performance of a Spring Boot application?

> Layer your answer: (1) profile first --- don't guess, use APM (e.g., New Relic, Datadog) or a profiler to find the actual bottleneck. (2) Common culprits at the app layer: N+1 queries, missing DB indexes, synchronous calls that could be async/parallelized, oversized thread pools causing context-switch overhead or undersized ones causing queueing. (3) JVM-level: right-size heap, pick the GC matching your latency/throughput needs, watch for excessive object allocation. (4) Connection pool tuning (HikariCP) --- undersized pools cause queueing under load, oversized ones waste DB resources. (5) Caching for expensive/repeated reads (Redis, Caffeine) with a clear invalidation strategy --- caching without invalidation strategy just becomes a stale-data bug factory.

### Q24. What are the key considerations when designing a microservices architecture?

> Service boundaries around business capability (not technical layers), data ownership (each service owns its data, no shared DB), communication strategy (sync vs async per interaction, not dogmatically all-one-way), resiliency patterns (timeouts, circuit breakers, retries with backoff) since network calls *will* fail, observability built in from day one (distributed tracing, centralized logging, metrics) because debugging across N services without it is nearly impossible, and deployment/operational overhead --- more services means more CI/CD pipelines, more monitoring surface, more operational complexity, so the split has to earn its complexity cost, not be done by default.

### Q25. How do you ensure the security of a Spring Boot application?

> Layered: authentication/authorization via Spring Security (JWT/OAuth2 for APIs), input validation (`@Valid` + Bean Validation annotations) to reject malformed/malicious input early, parameterized queries everywhere (no string-concatenated SQL), HTTPS/TLS enforced at the gateway/load balancer, secrets in a vault not in config files or source control, dependency scanning (OWASP Dependency-Check or Snyk) since a huge share of real vulnerabilities come from outdated libraries not your own code, rate limiting to blunt brute-force/DoS attempts, and security headers (CSRF protection, `X-Content-Type-Options`, CSP) which Spring Security gives you mostly by default.

## PART 3 --- Coding Questions

### 1. Check whether a number is a palindrome

    public class PalindromeNumber {
        public static boolean isPalindrome(int num) {
            if (num < 0) return false;       // negative numbers: treat as not palindrome
            int original = num;
            int reversed = 0;
            while (num != 0) {
                int digit = num % 10;
                reversed = reversed * 10 + digit;
                num /= 10;
            }
            return original == reversed;
        }

        public static void main(String[] args) {
            System.out.println(isPalindrome(12321));  // true
            System.out.println(isPalindrome(12345));  // false
        }
    }

**Complexity:** O(log n) time (number of digits), O(1) space.

### 2. Longest common substring between two strings

    public class LongestCommonSubstring {
        public static String find(String s1, String s2) {
            int m = s1.length(), n = s2.length();
            int[][] dp = new int[m + 1][n + 1]; // dp[i][j] = length of common substring ending at s1[i-1], s2[j-1]
            int maxLen = 0, endIndex = 0;

            for (int i = 1; i <= m; i++) {
                for (int j = 1; j <= n; j++) {
                    if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                        dp[i][j] = dp[i - 1][j - 1] + 1;
                        if (dp[i][j] > maxLen) {
                            maxLen = dp[i][j];
                            endIndex = i; // end index in s1
                        }
                    } // else dp[i][j] stays 0 — substring must be CONTIGUOUS
                }
            }
            return s1.substring(endIndex - maxLen, endIndex);
        }

        public static void main(String[] args) {
            System.out.println(find("abcdef", "zcdemf")); // "cde"
        }
    }

**Complexity:** O(m × n) time and space. Note this is *substring* (contiguous) --- different from *subsequence* (not necessarily contiguous), a common interview mix-up.

### 3. Check balanced parentheses

    import java.util.*;

    public class ValidParentheses {
        public static boolean isValid(String s) {
            Deque<Character> stack = new ArrayDeque<>();
            Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');

            for (char c : s.toCharArray()) {
                if (c == '(' || c == '[' || c == '{') {
                    stack.push(c);
                } else if (pairs.containsKey(c)) {
                    if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                        return false; // mismatched or unbalanced closing bracket
                    }
                }
            }
            return stack.isEmpty(); // true only if every opener was matched and closed
        }

        public static void main(String[] args) {
            System.out.println(isValid("({[]})"));  // true
            System.out.println(isValid("({[}])"));  // false
            System.out.println(isValid("(("));       // false — unclosed
        }
    }

**Complexity:** O(n) time, O(n) space worst case.

### 4. Second largest number in an array

    public class SecondLargest {
        public static int findSecondLargest(int[] arr) {
            if (arr.length < 2) throw new IllegalArgumentException("Need at least 2 elements");

            int largest = Integer.MIN_VALUE, secondLargest = Integer.MIN_VALUE;
            for (int num : arr) {
                if (num > largest) {
                    secondLargest = largest;
                    largest = num;
                } else if (num > secondLargest && num != largest) {
                    secondLargest = num;
                }
            }
            if (secondLargest == Integer.MIN_VALUE) {
                throw new IllegalArgumentException("No second distinct largest element found");
            }
            return secondLargest;
        }

        public static void main(String[] args) {
            System.out.println(findSecondLargest(new int[]{12, 35, 1, 10, 34, 1})); // 34
        }
    }

**Complexity:** O(n) time, O(1) space --- single pass, no sorting needed (sorting would be O(n log n), a common but suboptimal approach interviewers watch for).

### 5. Convert digits to words

    public class DigitsToWords {
        private static final String[] WORDS = {
            "Zero", "One", "Two", "Three", "Four", "Five", "Six", "Seven", "Eight", "Nine"
        };

        public static String convert(String digits) {
            StringBuilder result = new StringBuilder();
            for (char c : digits.toCharArray()) {
                if (Character.isDigit(c)) {
                    result.append(WORDS[c - '0']).append(" ");
                }
            }
            return result.toString().trim();
        }

        public static void main(String[] args) {
            System.out.println(convert("4092")); // "Four Zero Nine Two"
        }
    }

**Note:** this is "digit-by-digit" conversion (e.g., for reading out a phone number or account number digit by digit). If the question meant "convert a number's *value* to words" (e.g., 4092 → "Four thousand ninety two"), that's a meaningfully bigger problem involving place-value grouping --- worth clarifying with the interviewer which one they mean.

### 6. Implement a CRUD service in Spring Boot

    // Entity
    @Entity
    @Table(name = "products")
    public class Product {
        @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
        private String name;
        private BigDecimal price;
        // getters, setters, constructors omitted for brevity
    }

    // Repository
    public interface ProductRepository extends JpaRepository<Product, Long> {
    }

    // Service
    @Service
    public class ProductService {
        private final ProductRepository repository;

        public ProductService(ProductRepository repository) {
            this.repository = repository;
        }

        public Product create(Product product) {
            return repository.save(product);
        }

        public Product getById(Long id) {
            return repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found"));
        }

        public List<Product> getAll() {
            return repository.findAll();
        }

        public Product update(Long id, Product updated) {
            Product existing = getById(id);
            existing.setName(updated.getName());
            existing.setPrice(updated.getPrice());
            return repository.save(existing); // JPA detects this as an update, not insert, since ID is set
        }

        public void delete(Long id) {
            if (!repository.existsById(id)) {
                throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found");
            }
            repository.deleteById(id);
        }
    }

    // Controller
    @RestController
    @RequestMapping("/api/products")
    public class ProductController {
        private final ProductService service;

        public ProductController(ProductService service) {
            this.service = service;
        }

        @PostMapping
        public ResponseEntity<Product> create(@RequestBody Product product) {
            return ResponseEntity.status(HttpStatus.CREATED).body(service.create(product));
        }

        @GetMapping("/{id}")
        public Product getById(@PathVariable Long id) {
            return service.getById(id);
        }

        @GetMapping
        public List<Product> getAll() {
            return service.getAll();
        }

        @PutMapping("/{id}")
        public Product update(@PathVariable Long id, @RequestBody Product product) {
            return service.update(id, product);
        }

        @DeleteMapping("/{id}")
        public ResponseEntity<Void> delete(@PathVariable Long id) {
            service.delete(id);
            return ResponseEntity.noContent().build();
        }
    }

### 7. Rotate a string

    public class RotateString {

        // Rotate left by k positions
        public static String rotateLeft(String s, int k) {
            if (s.isEmpty()) return s;
            k = k % s.length(); // handle k larger than string length
            return s.substring(k) + s.substring(0, k);
        }

        // Rotate right by k positions
        public static String rotateRight(String s, int k) {
            if (s.isEmpty()) return s;
            k = k % s.length();
            return s.substring(s.length() - k) + s.substring(0, s.length() - k);
        }

        // Bonus: check if string B is a rotation of string A — classic follow-up question
        public static boolean isRotation(String a, String b) {
            if (a.length() != b.length()) return false;
            return (a + a).contains(b); // trick: every rotation of 'a' is a substring of 'a'+'a'
        }

        public static void main(String[] args) {
            System.out.println(rotateLeft("abcdef", 2));   // "cdefab"
            System.out.println(rotateRight("abcdef", 2));  // "efabcd"
            System.out.println(isRotation("waterbottle", "erbottlewat")); // true
        }
    }

**Complexity:** O(n) time and space for the basic rotation; the `isRotation` check is O(n) as well thanks to the concatenation trick.

## Final Prep Notes for This Round

-   This question set strongly signals a **Mastercard-style** round: heavy on transaction integrity (propagation, isolation, idempotency) because that's literally their domain. Be ready to tie *every* Spring/Java answer back to a payments example even if not asked directly --- it shows domain awareness.
-   For the coding questions, narrate your approach **before** coding (brute force first, then optimize) --- interviewers grade your reasoning process as much as the final code.
-   Practice the "why Mastercard" and "tell me about a challenge" answers out loud, not just in your head --- they sound very different once spoken. \# Caching --- In Depth \### Architecture, Patterns, Consistency, Failure Modes --- with Spring Boot/Redis Examples

## 1. Why Caching Exists (the actual trade-off)

A cache trades **consistency/freshness** for **speed/reduced load**. Every caching decision is really answering: *"How stale can this data be before it causes a real problem, and what happens when the cache and the source of truth disagree?"* If you can't answer both halves of that question for a given piece of data, you're not ready to cache it yet --- that framing alone is a strong thing to lead with in an interview.

## 2. Cache Levels --- Where It Sits in the Stack

    Browser Cache → CDN/Edge → Reverse Proxy/Gateway → App-Level Cache (local) → 
                                                          Distributed Cache (Redis) → Database (buffer pool) → Disk

Each layer you hit *before* reaching the database is a layer that protects the database from load and reduces latency for the user. A well-designed system fails the request "down" through these layers only when necessary.

  ------------------------------------------------------------------------------------------------------------------
  Layer             Example                   Typical TTL        Good for
  ----------------- ------------------------- ------------------ ---------------------------------------------------
  Browser           `Cache-Control` headers   minutes--days      Static assets, rarely-changing API responses

  CDN               CloudFront, Cloudflare    minutes--days      Images, JS/CSS, public API responses

  Reverse proxy     Nginx, Varnish            seconds--minutes   Full HTTP response caching for identical requests

  App-local         Caffeine, Guava           seconds--minutes   Hot, frequently-read, instance-tolerant data

  Distributed       Redis, Memcached          seconds--hours     Shared state across instances, session data

  DB buffer pool    InnoDB buffer pool        managed by DB      Automatic, not something you control directly
  ------------------------------------------------------------------------------------------------------------------

## 3. Local (In-Process) Cache --- Caffeine

Lives inside your app's JVM heap. Zero network latency, but **not shared** across instances --- if you run 5 pods, each has its own independent cache that can diverge.

    @Configuration
    public class CacheConfig {

        @Bean
        public Cache<String, Product> productCache() {
            return Caffeine.newBuilder()
                .maximumSize(10_000)                      // evicts via LRU-ish policy once full
                .expireAfterWrite(Duration.ofMinutes(10))  // TTL from write time
                .expireAfterAccess(Duration.ofMinutes(5))  // also expire if untouched for 5 min
                .recordStats()                             // exposes hit/miss metrics — always enable in prod
                .build();
        }
    }

    @Service
    public class ProductService {
        private final Cache<String, Product> cache;
        private final ProductRepository repository;

        public Product getProduct(String id) {
            return cache.get(id, key -> repository.findById(key)
                .orElseThrow(() -> new ProductNotFoundException(key)));
            // Caffeine handles the "check cache, else load and populate" logic atomically —
            // avoids a thundering-herd race where 100 threads all miss simultaneously and hit the DB
        }
    }

**When to use it:** read-heavy, latency-critical lookups where slight staleness across instances is tolerable --- e.g., feature flags, reference/lookup data (currency codes, country lists), or as an L1 cache in front of a distributed L2 cache (Redis) for the absolute hottest keys.

## 4. Distributed Cache --- Redis via Spring

Shared across every instance of your service. Slightly slower than local (network round trip, typically sub-millisecond to a few ms) but consistent fleet-wide and survives individual instance restarts.

### Declarative caching with `@Cacheable`

    @Configuration
    @EnableCaching
    public class RedisCacheConfig {

        @Bean
        public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
            RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()  // important — see "caching nulls" pitfall below
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));

            return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withCacheConfiguration("products", config.entryTtl(Duration.ofMinutes(30))) // per-cache override
                .build();
        }
    }

    @Service
    public class ProductService {

        @Cacheable(value = "products", key = "#id")
        public Product getProduct(String id) {
            // Only runs on a cache MISS. On hit, Redis value is returned directly, method body skipped.
            return productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
        }

        @CachePut(value = "products", key = "#product.id")
        public Product updateProduct(Product product) {
            // ALWAYS runs the method AND updates the cache with the return value — used for writes
            return productRepository.save(product);
        }

        @CacheEvict(value = "products", key = "#id")
        public void deleteProduct(String id) {
            productRepository.deleteById(id);
            // explicitly removes the stale entry so the next read repopulates from DB
        }

        @CacheEvict(value = "products", allEntries = true)
        public void clearAllProductCache() {
            // nuclear option — use sparingly, e.g. after a bulk import job
        }
    }

### Manual control with `RedisTemplate` (when you need more than annotations give you)

    @Service
    public class SessionService {
        private final RedisTemplate<String, Object> redisTemplate;

        public void storeSession(String sessionId, SessionData data) {
            redisTemplate.opsForValue().set(
                "session:" + sessionId,
                data,
                Duration.ofMinutes(30)   // explicit TTL per call, useful when it varies by context
            );
        }

        public Optional<SessionData> getSession(String sessionId) {
            SessionData data = (SessionData) redisTemplate.opsForValue().get("session:" + sessionId);
            return Optional.ofNullable(data);
        }

        public boolean tryAcquireLock(String key, String value, Duration ttl) {
            // Distributed lock pattern using Redis SETNX — common in payment idempotency checks
            Boolean acquired = redisTemplate.opsForValue().setIfAbsent(key, value, ttl);
            return Boolean.TRUE.equals(acquired);
        }
    }

**When to use distributed over local:** anything that must be consistent across instances (session data in a load-balanced fleet, rate-limit counters, idempotency keys, leaderboard-style data), or data too large to comfortably duplicate in every instance's heap.

## 5. Caching Strategies --- Read Patterns

### Cache-Aside (Lazy Loading) --- by far the most common

    public Product getProduct(String id) {
        Product cached = cache.get(id);
        if (cached != null) {
            return cached;                          // cache hit
        }
        Product fromDb = productRepository.findById(id).orElseThrow();
        cache.put(id, fromDb);                       // populate on miss
        return fromDb;
    }

The application owns the logic: check cache, fall back to DB on miss, populate cache. Simple, and only caches data that's actually requested --- no wasted cache space on data nobody reads. Downside: the *first* request for any key always pays the full DB latency (cold cache miss), and there's a brief window of inconsistency if the underlying data changes without an explicit eviction.

### Read-Through

The cache sits *in front of* the data source and handles loading transparently --- the application only ever talks to the cache, never the DB directly. Caffeine's `cache.get(key, loaderFunction)` shown earlier is effectively read-through. Architecturally cleaner (app code doesn't know the DB exists), but you're more dependent on the caching library/infra supporting it natively.

## 6. Caching Strategies --- Write Patterns

### Write-Through

Every write goes to the cache **and** the DB synchronously, as one logical operation, before the write is considered complete.

    public Product updateProduct(Product product) {
        Product saved = productRepository.save(product);  // DB write
        cache.put(saved.getId(), saved);                  // cache write, same operation
        return saved;
    }

**Trade-off:** cache and DB never disagree, but every write pays both costs --- slower writes. Good when reads vastly outnumber writes and consistency matters (e.g., product catalog).

### Write-Behind (Write-Back)

Write to cache immediately, return success to the caller, and persist to the DB **asynchronously** afterward (via a queue or background flush).

    public void updateProductFast(Product product) {
        cache.put(product.getId(), product);          // immediate, fast
        writeQueue.offer(product);                     // async worker flushes to DB later
    }

**Trade-off:** very fast writes, but a real risk of data loss if the app crashes before the async flush completes --- **never use this for financial/payment data** without a durable, replayable queue (e.g., Kafka) backing the async write, and even then it needs careful design. Good for things like view counters or analytics events where occasional loss is acceptable.

### Write-Around

Write goes directly to the DB, **bypassing the cache entirely**. The cache only gets populated later, lazily, on the next read (cache-aside style).

    public Product createProduct(Product product) {
        return productRepository.save(product);  // cache not touched at all here
        // next GET for this product will be a cache miss, populate cache then
    }

**Trade-off:** avoids polluting the cache with data that's written once and rarely re-read soon after (e.g., bulk imports, audit logs) --- but the first read after a write is always a guaranteed miss.

## 7. Eviction Policies

  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Policy                  Logic                                                      Use case
  ----------------------- ---------------------------------------------------------- ------------------------------------------------------------------------------------------------------------------------------------------------
  **LRU**                 Evict the entry not accessed for the longest time          Default choice --- works well for typical "recently used = likely to be used again" access patterns

  **LFU**                 Evict the entry accessed the fewest total times            Better when popularity matters more than recency --- a perennially popular item shouldn't get evicted just because it had one quiet hour

  **FIFO**                Evict oldest-inserted entry regardless of access pattern   Simple, rarely optimal, sometimes used for strictly time-ordered data

  **TTL-based**           Evict purely on elapsed time, regardless of access         Data with a known staleness tolerance --- exchange rates, weather data, feature flags

  **Random**              Evict a random entry                                       Surprisingly used in some systems (Redis supports `allkeys-random`) --- cheap to compute, avoids LRU's bookkeeping overhead at very high scale
  -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    # Configuring Redis's own eviction policy (when Redis itself hits maxmemory)
    maxmemory 2gb
    maxmemory-policy allkeys-lru   # other options: allkeys-lfu, volatile-lru, volatile-ttl, noeviction

`volatile-*` policies only evict keys that have a TTL set, leaving keys without a TTL untouched --- useful if you're mixing genuinely permanent keys with temporary cache entries in the same Redis instance.

## 8. Cache Invalidation --- "The Two Hard Problems in Computer Science"

This is the part that actually separates a junior answer from a senior one.

**Time-based (TTL)** --- simplest, but a blunt instrument: data can be stale for up to the full TTL window, and you're always trading "how fresh" against "how much load you're willing to put back on the DB."

**Event-based invalidation** --- explicitly evict/update the cache when the underlying data changes, rather than waiting for TTL.

    @Service
    public class ProductService {

        @CacheEvict(value = "products", key = "#product.id")
        @Transactional
        public void updateProduct(Product product) {
            productRepository.save(product);
            // eviction happens after the transaction commits successfully —
            // see "cache-DB consistency" section below for why ordering matters here
        }
    }

**Event-driven invalidation across services** --- in a microservices setup, Service B's cache of data owned by Service A needs to be invalidated when A's data changes. This is exactly where Kafka comes back in: A publishes a `ProductUpdated` event, B consumes it and evicts/refreshes its local cache entry --- decoupled, no direct coupling between A and B's cache implementations.

    @KafkaListener(topics = "product-events")
    public void onProductEvent(ProductUpdatedEvent event) {
        redisTemplate.delete("products::" + event.getProductId());
        // next read on this service repopulates from a fresh call to the product service
    }

**Versioned/keyed invalidation** --- instead of explicitly deleting, embed a version in the cache key (`product:123:v7`) so a data change just means callers start requesting a new key, and old versions naturally age out via TTL without needing a synchronous delete at all. Useful when explicit invalidation across many distributed caches is operationally hard to coordinate reliably.

## 9. The Big Failure Modes (these are what interviewers actually probe)

### Cache Stampede / Thundering Herd

A popular key expires, and a flood of concurrent requests all miss the cache at the same instant and hammer the DB simultaneously trying to repopulate it.

    // Vulnerable: every concurrent miss independently queries the DB
    public Product getProduct(String id) {
        Product cached = cache.get(id);
        if (cached == null) {
            cached = productRepository.findById(id).orElseThrow(); // 1000 threads can reach here at once
            cache.put(id, cached);
        }
        return cached;
    }
    // Fixed: lock per-key so only ONE thread repopulates, others wait for the result
    private final Map<String, Object> locks = new ConcurrentHashMap<>();

    public Product getProduct(String id) {
        Product cached = cache.get(id);
        if (cached != null) return cached;

        Object lock = locks.computeIfAbsent(id, k -> new Object());
        synchronized (lock) {
            cached = cache.get(id);             // re-check — another thread may have just populated it
            if (cached != null) return cached;
            cached = productRepository.findById(id).orElseThrow();
            cache.put(id, cached);
            return cached;
        }
    }

Caffeine's `cache.get(key, loaderFunction)` shown earlier handles this automatically --- it's one of the strongest reasons to use a real caching library instead of a raw `Map`. For Redis-backed caches, the equivalent is a distributed lock (`SETNX`) around the repopulation, or staggering TTLs with jitter so many keys don't expire at the exact same moment.

### Cache Penetration

Requests for a key that **doesn't exist in the DB at all** --- since it's never cached (there's nothing to cache), every such request bypasses the cache and hits the DB every time. An attacker can exploit this deliberately to overload your DB by requesting many non-existent IDs.

    // Fix: cache the "not found" result too, with a short TTL
    public Product getProduct(String id) {
        Product cached = cache.get(id);
        if (cached != null) {
            if (cached == NULL_PLACEHOLDER) return null; // cached negative result
            return cached;
        }
        Optional<Product> fromDb = productRepository.findById(id);
        if (fromDb.isEmpty()) {
            cache.put(id, NULL_PLACEHOLDER, Duration.ofSeconds(30)); // short TTL, prevents repeated DB hits
            return null;
        }
        cache.put(id, fromDb.get());
        return fromDb.get();
    }

A Bloom filter in front of the cache is the more scalable version of this fix --- it can definitively say "this key has never existed" in O(1) space-efficient fashion, rejecting the request before even checking the cache or DB.

### Cache Avalanche

A large number of cache entries expire **at the same time** (e.g., they were all populated together with the same TTL), causing a sudden mass simultaneous load on the DB --- like a stampede, but across many keys at once rather than one hot key.

**Fix:** add random jitter to TTLs so entries don't all expire in the same instant.

    Duration ttl = Duration.ofMinutes(10).plusSeconds(new Random().nextInt(120)); // +0-120s jitter
    cache.put(key, value, ttl);

### Stale Cache / Cache-DB Inconsistency

The classic ordering bug: if you update the DB *then* the cache write fails, or you evict the cache *before* the DB transaction actually commits, readers can see stale or even nonsensical data.

    // BUG: evicting before the transaction commits — a reader could repopulate the cache
    // with the OLD value if they read between the evict and the actual commit
    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public void updateProduct(Product product) {
        productRepository.save(product);
    }

The safer pattern is to evict **after** the transaction commits, not as part of it --- Spring's `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` is built for exactly this:

    @Transactional
    public void updateProduct(Product product) {
        productRepository.save(product);
        eventPublisher.publishEvent(new ProductUpdatedEvent(product.getId()));
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleProductUpdated(ProductUpdatedEvent event) {
        redisTemplate.delete("products::" + event.getProductId());
        // only evicts once we're CERTAIN the DB change is durable
    }

## 10. Caching Nulls --- A Subtle Pitfall

By default, many Spring cache setups will happily cache a `null` return value, meaning if a lookup genuinely fails once, every subsequent call returns the cached `null` until TTL expiry --- even after the underlying record actually gets created.

    @Cacheable(value = "products", unless = "#result == null")
    public Product getProduct(String id) {
        return productRepository.findById(id).orElse(null);
        // 'unless' prevents caching the null — combine with the cache-penetration fix above
        // if you specifically WANT to cache negative results with a short, deliberate TTL
    }

## 11. Cache Key Design

Bad key design causes silent bugs --- collisions across unrelated data, or accidental cache-wide invalidation.

    // Bad: ambiguous, could collide across entity types
    cache.put(id, product);

    // Good: namespaced, type-safe, includes any relevant context (tenant, version, locale)
    String key = "product:" + tenantId + ":" + productId + ":v2";
    cache.put(key, product);

With Spring's SpEL-based keys, be deliberate about composite keys:

    @Cacheable(value = "products", key = "#tenantId + ':' + #productId")
    public Product getProduct(String tenantId, String productId) { ... }

## 12. Monitoring a Cache (don't fly blind)

The metrics that actually matter: **hit ratio** (low ratio means the cache isn't earning its complexity --- investigate TTL, key design, or whether this data should be cached at all), **eviction rate** (high eviction under normal load means undersized cache), and **latency** (a distributed cache that's slower than just hitting the DB directly has failed its one job --- this happens more often than people expect under network issues or Redis contention).

    Cache<String, Product> cache = Caffeine.newBuilder()
        .recordStats()
        .build();

    CacheStats stats = cache.stats();
    double hitRate = stats.hitRate();        // expose this via Micrometer/Actuator to Prometheus/Grafana
    long evictionCount = stats.evictionCount();

## 13. Senior-Level Talking Points (what to actually say in the interview)

-   **Lead with the question, not the answer:** "How stale can this be, and what's the blast radius if the cache is wrong?" before jumping into Redis vs Caffeine.
-   **Never cache financial balances with a loose TTL.** Account balances, available credit, payment status --- these need either no caching, very short TTL with event-driven invalidation, or a cache-aside pattern where you accept the first-read cost rather than risk showing a stale balance.
-   **Distinguish "cache as optimization" from "cache as source of truth."** A cache should always be reconstructable from the real source of truth --- if losing the cache entirely (Redis goes down) would cause actual data loss, you've built a database, not a cache, and that's a design smell.
-   **Always mention the failure modes unprompted** (stampede, penetration, avalanche, staleness) --- this is the single biggest signal of real production experience vs. textbook knowledge on this topic.
-   **Tie back to your stack:** in a Kafka-driven microservices architecture, event-driven cache invalidation (consume a domain event, evict the relevant key) is almost always the right answer over polling or pure TTL --- it shows you're connecting caching to the rest of your architecture, not treating it as an isolated topic.
