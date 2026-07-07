# Senior Java Developer (10+ YOE) — Interview Q&A Study Guide

---

## Core Java

### 1. `==` vs `.equals()` — why is overriding `equals()` without `hashCode()` dangerous?
`==` compares references (memory addresses) for objects, and actual values for primitives. `.equals()` compares logical equality, as defined by the class.

If you override `equals()` but not `hashCode()`, you break the contract that **equal objects must have equal hash codes**. This means two "equal" objects could land in different buckets in a `HashMap`/`HashSet`, so lookups, `contains()`, and deduplication silently fail — the object won't be found even though `.equals()` would return `true`.

### 2. The `equals()`/`hashCode()` contract
- If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** be true.
- The reverse is not required — unequal objects *can* share a hash code (collision), just not the other way around.
- `hashCode()` must be consistent (same value across calls, unless fields used in the calculation change).

Violating this breaks hash-based collections silently — no exception is thrown, you just get wrong behavior (duplicates in a `Set`, missing keys in a `Map`).

### 3. `String` vs `StringBuilder` vs `StringBuffer`
- **`String`**: immutable. Every concatenation creates a new object — expensive in loops.
- **`StringBuilder`**: mutable, not thread-safe, fast — use for single-threaded string building (e.g., inside a loop).
- **`StringBuffer`**: mutable, thread-safe (synchronized methods) — use only if multiple threads mutate the same buffer concurrently; otherwise it's just overhead compared to `StringBuilder`.

### 4. Why is `String` immutable?
- **Security**: strings are used for class names, file paths, network connections — mutability would be a huge attack vector.
- **Caching/String pool**: the JVM can intern and reuse `String` literals since they can't change.
- **Thread-safety**: immutable objects are inherently safe to share across threads with no synchronization.
- **Hashcode caching**: since it can't change, `String` caches its hash code, making it a fast `HashMap` key.

### 5. Checked vs unchecked exceptions
- **Checked** (`extends Exception`): must be declared or caught at compile time. Use for *recoverable* conditions the caller should be forced to handle (e.g., `IOException` — file might not exist, caller should decide what to do).
- **Unchecked** (`extends RuntimeException`): not enforced by the compiler. Use for programming errors / invariant violations that shouldn't normally be recovered from at the call site (e.g., `IllegalArgumentException`, `NullPointerException`).

In modern practice, many teams lean toward unchecked exceptions even for business logic errors, since checked exceptions tend to pollute method signatures and get caught-and-ignored anyway. Good answer: "it depends on whether the caller has a meaningful way to recover."

### 6. `final`, `finally`, `finalize()`
- **`final`**: keyword — a `final` variable can't be reassigned, a `final` method can't be overridden, a `final` class can't be extended.
- **`finally`**: block that always executes after a `try` (whether or not an exception was thrown), typically used for cleanup (closing resources) — largely superseded by try-with-resources.
- **`finalize()`**: method called by the GC before reclaiming an object — **deprecated since Java 9**, unreliable timing, avoid entirely; use `try-with-resources` or `Cleaner` instead.

### 7. Abstract classes vs interfaces
- **Abstract class**: can hold state (instance fields), constructors, and a mix of implemented/abstract methods. A class can extend only one.
- **Interface**: traditionally only method signatures + constants; a class can implement many.
- **Java 8+ default methods** blurred the line — interfaces can now provide implementation. But interfaces still can't hold instance state (only `static final` constants), so the core distinction (single inheritance of state vs. multiple inheritance of behavior) still holds. Practical rule of thumb: use interfaces to define a *contract/capability* (`Comparable`, `Runnable`); use abstract classes when subclasses share actual state or a common partial implementation.

### 8. Autoboxing/unboxing pitfalls
`Integer` caches instances for values **-128 to 127** (`Integer.valueOf()` uses this cache). So `Integer a = 127; Integer b = 127;` — both point to the same cached object, `a == b` is `true`. Outside that range (e.g., `200`), new objects are created each time, so `a == b` is `false` even though `.equals()` is `true`.

**Lesson**: never use `==` to compare boxed types — always use `.equals()`, or unbox to primitives first.

---

## Collections

### 9. How `HashMap` works internally
- Backed by an array of "buckets" (`Node<K,V>[] table`).
- `hash(key)` computes a hash, which is spread (XORed with itself shifted right 16 bits) to reduce collisions, then `& (table.length - 1)` maps it to a bucket index.
- Each bucket holds a linked list of entries with the same index (collisions).
- **Java 8+**: if a bucket's linked list grows beyond a threshold (8 entries) *and* the table is large enough, it's converted to a red-black tree for that bucket — turning worst-case O(n) lookup into O(log n).
- **Resizing**: when `size > capacity * loadFactor` (default load factor 0.75), the table doubles in size and all entries are rehashed — an expensive but amortized O(1) operation.

### 10. `HashMap` vs `LinkedHashMap` vs `TreeMap`
- **`HashMap`**: no ordering guarantee, O(1) average operations.
- **`LinkedHashMap`**: maintains insertion order (or access order, if configured) via an internal doubly-linked list — useful for LRU cache implementations.
- **`TreeMap`**: sorted by key (natural ordering or a `Comparator`), backed by a red-black tree — O(log n) operations, useful when you need range queries or sorted iteration.

### 11. `ArrayList` vs `LinkedList`
- **`ArrayList`**: backed by a dynamic array. O(1) random access (`get(i)`), O(n) insert/delete in the middle (shifting), amortized O(1) append.
- **`LinkedList`**: doubly-linked list. O(1) insert/delete at known nodes (e.g., head/tail), but O(n) random access since you must traverse.
- In practice, `ArrayList` is almost always the better default — better cache locality, lower memory overhead per element. `LinkedList` mainly makes sense for queue/deque use cases with frequent head/tail operations (though `ArrayDeque` usually wins there too).

### 12. Modifying a `List` while iterating
Structurally modifying a list (add/remove) while iterating with a standard iterator throws `ConcurrentModificationException` — the iterator tracks a `modCount` and detects the mismatch on `next()`.

**How to avoid it**:
- Use `Iterator.remove()` instead of `list.remove()`.
- Use `removeIf()`.
- Iterate over a copy: `new ArrayList<>(list)`.
- Use `CopyOnWriteArrayList` if concurrent modification is expected (trades memory/write-cost for safe iteration).

### 13. `Comparable` vs `Comparator`
- **`Comparable`**: implemented *by the class itself* (`compareTo()`) — defines a single "natural ordering."
- **`Comparator`**: a separate strategy object (`compare(a, b)`) — lets you define multiple, external, swappable orderings without modifying the class. Preferred when sorting the same objects different ways in different contexts.

### 14. Fail-fast vs fail-safe iterators
- **Fail-fast** (`ArrayList`, `HashMap`): throws `ConcurrentModificationException` if the collection is structurally modified during iteration — detected via a `modCount` check. Protects against silent bugs but isn't itself thread-safe.
- **Fail-safe** (`CopyOnWriteArrayList`, `ConcurrentHashMap`'s iterator): iterates over a snapshot or tolerates concurrent modification without throwing, but may not reflect the very latest state during iteration.

---

## Concurrency

### 15. Java Memory Model & `volatile`
The JMM defines how threads interact through memory — specifically, when writes by one thread become visible to another.

`volatile` guarantees:
- **Visibility**: writes to a volatile variable are immediately visible to all threads (no caching in a thread-local CPU register/cache).
- **Ordering**: prevents instruction reordering around the volatile access (establishes a happens-before relationship).

It does **not** guarantee atomicity for compound operations (e.g., `count++` on a volatile `int` is still a race condition — read, increment, write are three separate steps).

### 16. `synchronized` methods/blocks vs `ReentrantLock`
- **`synchronized`**: JVM-managed intrinsic lock. Simple, automatically released even on exception. Can't be interrupted while waiting, no timeout, no fairness policy, one condition per lock.
- **`ReentrantLock`**: explicit lock (`java.util.concurrent.locks`). Supports `tryLock()` with timeout, interruptible lock acquisition, fairness policy, and multiple `Condition` objects per lock. More flexible but you must remember to `unlock()` in a `finally` block.

Use `synchronized` for simple cases; reach for `ReentrantLock` when you need timeout/interruption/fairness/multiple wait conditions.

### 17. Deadlock
Occurs when two or more threads each hold a lock the other needs, and neither releases — circular wait.

**Example**: Thread A locks `resource1` then tries to lock `resource2`; Thread B locks `resource2` then tries to lock `resource1`. Both block forever.

**Prevention**:
- Always acquire locks in a **consistent global order**.
- Use `tryLock()` with a timeout instead of blocking indefinitely.
- Minimize lock scope and avoid nested locks where possible.

### 18. `ExecutorService` thread pool types
- **`FixedThreadPool`**: fixed number of threads, unbounded queue — predictable resource usage, but tasks can pile up in the queue under sustained load.
- **`CachedThreadPool`**: creates threads as needed, reuses idle ones, unbounded — good for many short-lived async tasks, but can spawn unbounded threads under heavy load (risk of resource exhaustion).
- **`ScheduledThreadPool`**: supports delayed and periodic task execution.

In production, it's common to build a custom `ThreadPoolExecutor` directly with a bounded queue and explicit rejection policy, rather than the `Executors` factory defaults — the defaults' unbounded queues/threads are a common source of production incidents.

### 19. `Callable` vs `Runnable`
- **`Runnable.run()`**: no return value, can't throw checked exceptions.
- **`Callable<V>.call()`**: returns a value `V`, can throw checked exceptions. Used with `ExecutorService.submit()` which returns a `Future<V>`.

### 20. `CompletableFuture`
Represents an async computation that can be composed/chained.
```java
CompletableFuture.supplyAsync(() -> fetchUser())
    .thenApply(user -> user.getName())
    .thenAccept(name -> System.out.println(name))
    .exceptionally(ex -> {
        log.error("failed", ex);
        return null;
    });
```
- `thenApply`: transform result (sync, on same thread by default).
- `thenApplyAsync`: same, but on the common ForkJoinPool (or a supplied executor).
- `thenCompose`: chain another async operation (flattens nested futures — like `flatMap`).
- `exceptionally` / `handle`: exception handling in the chain — exceptions propagate down the chain like a checked pipeline, similar to promise `.catch()` in JS.

### 21. `ConcurrentHashMap` internals
Achieves thread-safety without locking the whole map by using **lock striping** (historically, an array of segments each with its own lock; Java 8+ moved to finer-grained locking per-bin using `synchronized` on the head node of each bucket, combined with CAS operations for many operations). This means concurrent writes to *different* buckets don't block each other — much higher throughput than `Collections.synchronizedMap()`, which locks the entire map on every operation.

### 22. Producer-consumer with `BlockingQueue`
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(capacity);

// Producer
queue.put(task); // blocks if full

// Consumer
Task task = queue.take(); // blocks if empty
```
`BlockingQueue` handles all the wait/notify coordination internally — no manual `wait()`/`notify()` needed. Bounded queues (`LinkedBlockingQueue(capacity)`, `ArrayBlockingQueue`) also give you natural backpressure.

### 23. Race condition
Occurs when the correctness of a result depends on timing/interleaving of threads. Classic example: two threads both doing `count++` on a shared non-volatile, non-atomic `int` — the read-modify-write isn't atomic, so increments can be lost.

**Real-world example to mention**: a shared counter or cache being updated by multiple request-handling threads without synchronization, leading to occasional undercounts, or a "check-then-act" bug (e.g., checking `if (map.containsKey(k))` then `map.put(k, v)` — another thread can insert between the check and the act).

---

## JVM & Performance

### 24. JVM memory structure
- **Heap**: object storage, shared across threads, subject to GC. Split into Young Gen (Eden + Survivor spaces) and Old Gen.
- **Stack**: per-thread, stores local variables and method call frames — not GC'd, reclaimed automatically as frames pop.
- **Metaspace** (Java 8+, replaced PermGen): stores class metadata, method bytecode — allocated from native memory, grows dynamically (unlike the fixed-size PermGen, which commonly caused `OutOfMemoryError: PermGen space`).

### 25. Garbage Collection
Most objects die young (generational hypothesis), so the heap is split:
- **Young Gen**: new objects allocated in Eden. **Minor GC** runs frequently, copies surviving objects between two Survivor spaces; objects that survive several cycles get **promoted** to Old Gen.
- **Old Gen**: long-lived objects. **Major/Full GC** runs less often but is more expensive (scans the whole heap) — a common cause of latency spikes ("stop-the-world" pauses).

### 26. GC algorithms
- **Parallel GC**: multi-threaded, throughput-focused, stop-the-world pauses — good for batch jobs where pause time doesn't matter.
- **G1 (Garbage First)**: default since Java 9. Divides heap into regions, prioritizes collecting regions with the most garbage first, aims for predictable pause times — good general-purpose choice for most services.
- **ZGC**: designed for very large heaps with sub-millisecond pause targets, using colored pointers and concurrent processing — good for latency-sensitive, large-heap applications.

Choice depends on whether you're optimizing for **throughput** (Parallel) or **latency/pause time** (G1, ZGC).

### 27. Memory leaks despite GC
GC only reclaims objects with no reachable references — a "leak" in Java means objects are unintentionally still *reachable*, so GC can't touch them. Common causes:
- Static collections that grow indefinitely (e.g., a cache with no eviction).
- Listeners/callbacks registered but never unregistered.
- Inner classes holding implicit references to outer class instances.
- `ThreadLocal` values not cleaned up in pooled-thread environments.

### 28. Diagnosing production issues
- **High CPU**: `jstack` (or `top -H` + `jstack`) to get thread dumps, identify which threads are consuming CPU and what they're doing (busy loop? excessive GC? lock contention?).
- **Memory leak**: `jmap -histo` for object counts, heap dumps (`jmap -dump`) analyzed in tools like Eclipse MAT or VisualVM to find retained object graphs and dominators.
- **General approach**: reproduce if possible, gather dumps/metrics under load, correlate with logs/APM traces (e.g., New Relic, Datadog), form a hypothesis, verify with data — not guesswork.

---

## Java 8+ Features

### 29. Streams — intermediate vs terminal operations
- **Intermediate** (`filter`, `map`, `sorted`, `distinct`): lazy, return a new stream, don't execute until a terminal op is invoked.
- **Terminal** (`collect`, `forEach`, `reduce`, `count`): triggers actual execution of the whole pipeline, produces a result or side effect. A stream can only be consumed once.

### 30. `map()` vs `flatMap()`
- `map()`: transforms each element 1-to-1 (`Stream<T> → Stream<R>`).
- `flatMap()`: transforms each element into a stream, then flattens all those streams into one (`Stream<List<T>> → Stream<T>`) — useful when each input maps to *multiple* outputs, e.g., flattening a list of lists.

### 31. Lazy evaluation in streams
Intermediate operations don't run until a terminal operation pulls elements through. Example: `stream.filter(x -> {System.out.println("filtering " + x); return x > 5;}).findFirst()` — because of lazy evaluation and short-circuiting, this stops as soon as it finds a match, rather than filtering the entire collection first. This matters for performance on large or infinite streams (`Stream.iterate`).

### 32. Functional interfaces
An interface with exactly one abstract method (can have default/static methods too), enabling lambda syntax.
- **`Function<T,R>`**: takes T, returns R — transformation.
- **`Predicate<T>`**: takes T, returns boolean — filtering/condition.
- **`Supplier<T>`**: takes nothing, returns T — lazy value production (e.g., default values, factories).
- **`Consumer<T>`**: takes T, returns nothing — side-effecting action (e.g., logging, printing).

### 33. `Optional.of()` vs `.ofNullable()` vs `.empty()`
- **`Optional.of(value)`**: throws `NullPointerException` immediately if `value` is null — use when you're certain it's non-null (fail fast).
- **`Optional.ofNullable(value)`**: safely wraps a possibly-null value into `Optional.empty()` if null.
- **`Optional.empty()`**: explicit empty instance.

**Best practices**: use `Optional` as a *return type* to signal "may be absent," not as a field type or method parameter (adds needless overhead/complexity). Avoid `.get()` without checking `.isPresent()` — prefer `.orElse()`, `.orElseGet()`, `.orElseThrow()`, or `.map()`/`.ifPresent()` for a more functional style.

### 34. Default and static methods in interfaces
Introduced in Java 8 primarily to **evolve APIs without breaking existing implementations** — e.g., adding `forEach()` to `Collection` without forcing every existing implementing class to add it. Default methods provide a fallback implementation; static methods provide utility functions scoped to the interface (e.g., `Comparator.comparing(...)`).

---

## Design & Architecture

### 35. "Walk me through a system you designed"
Structure your answer: **Context → Constraints → Design → Trade-offs → Outcome.**
- What was the problem/scale?
- What constraints mattered (latency, consistency, cost, team size)?
- What did you choose, and what did you explicitly *not* choose, and why?
- What would you do differently now?

(This is candidate-specific — prepare 1-2 real examples from your own work in this structure.)

### 36. Designing for testability
- **Dependency Injection**: pass collaborators in via constructor rather than instantiating them internally — lets you substitute mocks/fakes in tests.
- **Program to interfaces**, not concrete classes, at integration boundaries (DB, external APIs, clock/time) so they can be mocked.
- Avoid static/singleton state that's hard to reset between tests.
- Separate pure business logic from I/O/side-effects so the logic can be unit tested without a real DB/network call.

### 37. SOLID principles (brief + example angle)
- **S**ingle Responsibility — a class should have one reason to change.
- **O**pen/Closed — open for extension, closed for modification (e.g., strategy pattern instead of a big if/else that needs editing for every new case).
- **L**iskov Substitution — subtypes must be substitutable for their base type without breaking correctness.
- **I**nterface Segregation — prefer several small, specific interfaces over one large one.
- **D**ependency Inversion — depend on abstractions, not concrete implementations (ties directly to DI in Q36).

Be ready with one concrete "here's where I applied/violated and fixed this" story from real code.

### 38. Design patterns actually used
Common senior-level answers: **Builder** (constructing complex immutable objects, e.g., request DTOs), **Strategy** (swappable algorithms, e.g., pricing rules), **Factory** (centralizing object creation logic, especially when it depends on config/runtime type), **Decorator** (adding behavior like logging/caching around a service without modifying it), **Observer** (event-driven systems, listeners). Avoid reciting textbook UML — describe a real scenario.

### 39. Backward compatibility for public APIs
- Add new fields/methods rather than changing existing signatures.
- Use versioning (URL path, header, or package versioning) when breaking changes are unavoidable.
- Deprecate before removing — give consumers a migration window (`@Deprecated` + docs + monitoring usage of the old path).
- For serialized data formats, ensure new fields have sensible defaults so old clients/consumers don't break on missing fields.

### 40. Designing a caching layer
- **Eviction policy**: LRU (most common, easy via `LinkedHashMap` with access order, or Caffeine/Guava cache), LFU, or TTL-based.
- **TTL**: expire entries after a fixed time to bound staleness.
- **Cache stampede prevention** (many requests missing simultaneously and hammering the DB): use a "lock/single-flight" pattern so only one thread recomputes a missing value while others wait, or serve slightly-stale data while recomputing in the background (stale-while-revalidate).
- Consider cache layer placement: in-process (fast, but per-instance, not shared) vs. distributed (Redis/Memcached — shared across instances, adds network hop).

---

## Behavioral

### 41. Debugging a production incident under pressure
Structure: **Detect → Triage → Mitigate → Root cause → Prevent.**
Emphasize: staying calm, communicating status to stakeholders, prioritizing mitigation (rollback/feature flag) over root-causing live, then doing a proper postmortem. Concrete example from your own experience is what interviewers actually want here — prepare one with real specifics (metrics, timeline, what you personally did).

### 42. Disagreeing with a technical decision
Interviewers are checking for **collaboration and maturity**, not "who won." Good structure: state your concern with data/reasoning, listen to the counter-reasoning, find where you actually disagree (values vs facts), and describe how it was resolved (compromise, deferring to more context, or escalating respectfully). Avoid answers where you were simply "right" and everyone agreed — show real disagreement-handling skill.

### 43. Mentoring / code review approach
Focus on: giving actionable, specific feedback (not vague), distinguishing "must-fix" from "nice-to-have," explaining *why* not just *what*, being consistent, and pairing/teaching rather than just correcting. Bonus: mention how you calibrate feedback style to the person's experience level.

---

## Study tips
- For concurrency and JVM questions, be ready to draw diagrams if this becomes an in-person/whiteboard-style round.
- For collections/Streams, actually write and run small snippets — muscle memory matters under interview pressure.
- For behavioral questions, prepare 3-4 flexible stories (incident, disagreement, mentoring, technical trade-off) you can adapt to different specific questions.
