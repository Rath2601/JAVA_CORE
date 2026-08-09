# Java Collections Framework — Production Reference Guide

> Baseline: **Java 17 (LTS)**. Implementation notes reflect JDK 9+ source where it diverges from Java 8.
>
> **Contents:** 1. List · 2. Set · 3. Queue & Deque · 4. Map · 5. Cross-Cutting Cheat Sheets

---

## 1. List Implementations

| Collection / Pattern | Thread-Safety | Complexity | Best Use Case / Workload | Internal Mechanics & Memory Behavior | Trade-offs & Anti-Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`ArrayList`** | No | • Append: Amortized $O(1)$<br>• Index Access: $O(1)$<br>• Insert/Remove at index: $O(N)$<br>• `contains`: $O(N)$ | Single-threaded default for lists, index lookups, and general ordered storage. **Default choice unless proven otherwise.** | Contiguous `Object[]` of **references**. `new ArrayList<>()` allocates a shared empty array; the 10-element backing array is created lazily on first `add`. Growth: `old + (old >> 1)` ($1.5\times$), via `Arrays.copyOf`. | Not thread-safe (fail-fast iterator → `ConcurrentModificationException`). Repeated growth = repeated $O(N)$ array copies; pre-size when $N$ is known. **Overload trap:** on `List<Integer>`, `remove(1)` removes *index* 1, `remove(Integer.valueOf(1))` removes the *value*. |
| **`LinkedList`** | No | • Head/Tail Insert/Remove: $O(1)$<br>• Positional Access: $O(N)$<br>• `contains`: $O(N)$ | Effectively none in modern production Java. `ArrayDeque` beats it as a Deque; `ArrayList` beats it as a List. | Doubly-linked `Node` objects scattered across the heap (`prev`, `next`, `item`). ~32–40 bytes per node vs. 4–8 bytes per `ArrayList` slot. | Abysmal cache behavior (pointer chasing defeats hardware prefetch). Even "$O(1)$ middle insert" is a lie in practice — you pay $O(N)$ to *traverse* to the position first. Implements both `List` and `Deque`, which invites misuse. |
| **`Collections.synchronizedList`** | Yes (single coarse mutex) | • Same as the wrapped list<br>• Plus uncontended lock overhead per call | Legacy interop; retrofitting safety onto an existing single-threaded structure with minimal code change. | Delegating wrapper; every method body is `synchronized (mutex) { ... }` on a single monitor. | **Anti-pattern under real concurrency** — one lock serializes all readers *and* writers. **Iteration is NOT protected:** you must manually `synchronized (list) { for (...) }` or you get a CME. Compound actions (`if (!list.contains(x)) list.add(x)`) are still racy. |
| **`CopyOnWriteArrayList`** | Yes (lock-free reads, locked writes) | • `get(i)`: $O(1)$<br>• `contains`/`indexOf`: $O(N)$<br>• Any write: $O(N)$ time **and** $O(N)$ allocation | Read-dominated concurrent state: event listeners, routing tables, feature flags, cached config. Rule of thumb: **>95% reads, small $N$**. | Writes clone the entire backing array under a `synchronized` block on an internal lock object (JDK 9+ replaced the older `ReentrantLock`). Readers dereference a `volatile Object[]` with **zero** synchronization. | **Anti-pattern for write-heavy or large collections** — severe GC churn from per-write array allocation. Iterator is a **point-in-time snapshot**: never throws CME, never observes later writes, and `iterator.remove()` / `set` throw `UnsupportedOperationException`. |
| **`List.of(...)` / `List.copyOf(...)`** | Yes (immutable) | • Index Access: $O(1)$ | Constants, defensive returns from APIs, DTO fields, method parameters that must not be mutated. | Compact `ImmutableCollections.ListN` (or `List12` for 1–2 elements) — no wrapper indirection, lower footprint than `ArrayList`. | Rejects `null` elements at construction. All mutators throw `UnsupportedOperationException`. `List.copyOf` takes a **defensive copy**; `Collections.unmodifiableList` is a **live read-only view** whose contents can still change beneath you. |
| **`Arrays.asList(...)`** | No | • Index Access: $O(1)$ | Quick fixed-size adapter over an existing array; varargs literals in tests. | Thin `Arrays$ArrayList` view **backed by the original array** — writes through `set()` mutate the source array. | Fixed-size: `add`/`remove` throw `UnsupportedOperationException`, but `set` succeeds. Frequently confused with `List.of` (immutable, null-hostile). `Arrays.asList(intArray)` yields a `List<int[]>` of size 1. |

---

## 2. Set Implementations

| Collection / Pattern | Thread-Safety | Complexity | Best Use Case / Workload | Internal Mechanics & Memory Behavior | Trade-offs & Anti-Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`HashSet`** | No | • Add/Remove/Contains: Average $O(1)$, Worst $O(\log N)$ | Single-threaded default for uniqueness and membership tests when order is irrelevant. | Backed by a `HashMap<E, Object>` with a shared dummy value (`PRESENT`). Hash is spread via `h ^ (h >>> 16)` because the bucket index is `hash & (n-1)` and only the low bits survive. Bins convert to red-black trees at **≥8 entries in a bin AND ≥64 table capacity** (below 64 it resizes instead). | Not thread-safe. The $O(\log N)$ worst case holds **only if elements implement `Comparable`** — otherwise the tree falls back to `identityHashCode` tie-breaking. Iteration order is unspecified and changes on resize. **Never mutate a field used in `hashCode()` after insertion** — the element becomes unreachable and leaks. Sizing on 17: `new HashSet<>((int)(n / 0.75f) + 1)`. |
| **`LinkedHashSet`** | No | • Add/Remove/Contains: Average $O(1)$ | Deduplication that must preserve **insertion order** — deterministic output, reproducible tests, ordered de-dup of a stream. | Extends `HashSet`; the backing map is a `LinkedHashMap` threading a doubly-linked list (`before`/`after`) through all entries. Iteration is $O(N)$ over the link chain, not over table capacity. | Two extra references per entry vs. `HashSet`. **Insertion-order only — there is no access-order constructor.** Access-ordering (and the `removeEldestEntry` LRU hook) exists on `LinkedHashMap`, not here. |
| **`TreeSet`** | No | • Add/Remove/Contains: $O(\log N)$<br>• `first`/`last`: $O(\log N)$ | Sorted iteration and range queries: `subSet`, `headSet`, `tailSet`, `floor`, `ceiling`, `higher`, `lower`, `descendingSet`. | Backed by a `TreeMap` (red-black tree). Ordering comes from `Comparable` or an explicit `Comparator`. | **Uniqueness is defined by `compare() == 0`, not by `equals()`.** A comparator inconsistent with equals silently drops elements. Throws `NPE` on `null` under natural ordering. $O(\log N)$ on every op with poor cache behavior. |
| **`EnumSet`** | No | • Add/Remove/Contains: $O(1)$, effectively a couple of instructions | Any set of enum constants: permissions, flags, state machines, feature toggles. **Always prefer over `HashSet<SomeEnum>`.** | Abstract; factory-only (`noneOf`, `of`, `allOf`, `range`, `complementOf`). `RegularEnumSet` packs ≤64 constants into a **single `long` bit vector**; `JumboEnumSet` uses a `long[]`. Bulk ops are bitwise AND/OR. | Not thread-safe. Rejects `null`. Iteration order is **natural enum declaration order**, not insertion order. No public constructor. |
| **`Set.of(...)` / `Set.copyOf(...)`** | Yes (immutable) | • Contains: $O(1)$ | Constants, allow-lists, defensive API returns. | `ImmutableCollections.SetN` — open-addressed probe table, no per-entry `Node` objects. | Rejects `null`. Throws `IllegalArgumentException` on **duplicate arguments** at construction. **Iteration order is randomized per JVM run** by a `SALT` value — deliberately, to break code that depends on order. |
| **`ConcurrentHashMap.newKeySet()`** | Yes (CAS + fine-grained `synchronized`) | • Add/Remove/Contains: Average $O(1)$ | **Default concurrent set.** High-concurrency membership, dedup caches, in-flight request tracking, idempotency keys. | Key view over a `ConcurrentHashMap` with `Boolean.TRUE` as value. Empty bins are populated with a lock-free CAS; contended bins `synchronize` on the **head node only**, so unrelated bins never contend. | Rejects `null`. Unordered. `size()` is an **estimate** derived from striped counters — never use it for exact invariants. Iterators are **weakly consistent** (may reflect writes made after creation) — a different guarantee from COW's snapshot. |
| **`ConcurrentSkipListSet`** | Yes (lock-free CAS) | • Add/Remove/Contains: $O(\log N)$<br>• `size()`: $O(N)$ | Concurrent **sorted** sets and concurrent range queries — the thread-safe `TreeSet`. | Backed by `ConcurrentSkipListMap`: a probabilistic multi-level linked list updated via atomic pointer CAS. Implements `NavigableSet`. | Rejects `null`. Higher per-node memory than `TreeSet`. **`size()` traverses the whole structure and is not atomic.** |
| **`CopyOnWriteArraySet`** | Yes (lock-free reads, locked writes) | • Contains: $O(N)$<br>• Write: $O(N)$ search + $O(N)$ copy | Very small ($N < \sim100$), read-dominated sets: listener registries, subscriber lists. | Backed by a `CopyOnWriteArrayList` using `addIfAbsent`. Uniqueness is a **linear `equals` scan**, not hashing. | **Double penalty per write.** Contains is $O(N)$ even for reads — unlike `CopyOnWriteArrayList`, there is no $O(1)$ path. Snapshot iterator; `remove()` unsupported. |
| **`Collections.synchronizedSet`** | Yes (single coarse mutex) | • Same as wrapped set + lock overhead | Legacy interop only. | Delegating wrapper synchronizing every method on one monitor. | Same failure modes as `synchronizedList`. Superseded by `ConcurrentHashMap.newKeySet()` in every respect. |

---

## 3. Queue & Deque Implementations

| Collection / Pattern | Thread-Safety | Complexity | Best Use Case / Workload | Internal Mechanics & Memory Behavior | Trade-offs & Anti-Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`ArrayDeque`** | No | • Enqueue/Dequeue at either end: Amortized $O(1)$<br>• Search/Remove(Object): $O(N)$ | Single-threaded default for **both Queue (FIFO) and Stack (LIFO)**. Replaces the legacy `Stack` and `LinkedList`. | Circular `Object[]` ring buffer with `head`/`tail` indices; default capacity 16. Zero per-element node allocation and excellent scan locality. On JDK 9+ growth is ~$2\times$ below 64 elements, $1.5\times$ above. | Not thread-safe. **Rejects `null`** (null is the sentinel for "empty"). No index-based random access. Note `Stack` iterates **bottom-to-top** — `ArrayDeque` used as a stack iterates top-to-bottom. |
| **`PriorityQueue`** | No | • Offer/Poll: $O(\log N)$<br>• Peek: $O(1)$<br>• `contains`/`remove(Object)`: $O(N)$<br>• Bulk construction: $O(N)$ | Top-$K$ problems, min/max heaps, scheduling, merge-$K$-sorted-lists — single-threaded. | Array-backed binary **min-heap**; `siftUp` on offer, `siftDown` on poll. Constructing from an existing `Collection` uses Floyd heapify in $O(N)$, not $N \log N$. Default capacity 11. | **Iteration order is NOT sorted** — only repeated `poll()` yields ordering, and `toString()` prints raw heap-array order. Rejects `null`. Unbounded (grows until OOM). Not thread-safe. |
| **`ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`** | Yes (lock-free CAS) | • Enqueue/Dequeue: $O(1)$ amortized<br>• `size()`: $O(N)$ | High-throughput **non-blocking** handoff where consumers must never block — work-stealing, metrics buffers, actor mailboxes. | Michael–Scott non-blocking algorithm: CAS on `volatile` node pointers, with lazy head/tail advancement. Nodes are allocated per element. | **Unbounded → no backpressure.** A consumer slower than the producer will OOM the JVM — the single most important production property. `size()` walks the whole chain and is only an estimate — use `isEmpty()`. Weakly consistent iterator. No blocking `take()`. |
| **`ArrayBlockingQueue`** | Yes (blocking, single lock) | • Offer/Poll/Put/Take: $O(1)$<br>• `size()`: $O(1)$ | Bounded producer–consumer with a hard memory ceiling and **zero allocation churn** — the safe default for a `ThreadPoolExecutor` work queue. | Pre-allocated bounded ring buffer guarded by **one** `ReentrantLock` with two `Condition`s (`notEmpty`, `notFull`). Optional fairness flag. | Capacity is fixed at construction and cannot resize. Producers and consumers contend on the **same** lock, capping throughput vs. `LinkedBlockingQueue`. Fair mode costs significant throughput. |
| **`LinkedBlockingQueue`** | Yes (blocking, dual lock) | • Offer/Poll/Put/Take: $O(1)$<br>• `size()`: $O(1)$ (`AtomicInteger`) | Higher-throughput bounded producer–consumer where producers and consumers should run in parallel. | Singly-linked nodes guarded by **two separate** `ReentrantLock`s (`putLock`, `takeLock`) plus an `AtomicInteger` count — a producer and a consumer can proceed simultaneously. | **Default capacity is `Integer.MAX_VALUE`.** This is why `Executors.newFixedThreadPool()` OOMs under sustained load, and why a `ThreadPoolExecutor` backed by an unbounded queue **never grows past `corePoolSize`** — `maximumPoolSize` is dead configuration. Always pass an explicit capacity. Higher GC churn than `ArrayBlockingQueue`. |
| **`LinkedBlockingDeque`** | Yes (blocking, single lock) | • Insert/Remove at either end: $O(1)$ | Bounded work-stealing pools; retry queues where failed items are pushed back to the head. | Doubly-linked nodes under a **single** `ReentrantLock` — the two-lock split is impossible when both ends can meet. Optionally bounded. | Loses the dual-lock throughput advantage of `LinkedBlockingQueue`. Also defaults to `Integer.MAX_VALUE` if capacity is omitted. |
| **`PriorityBlockingQueue`** | Yes (blocking on take only) | • Offer/Poll: $O(\log N)$<br>• Peek: $O(1)$ | Concurrent priority scheduling: tiered job queues, SLA-ordered task dispatch. | Array binary heap under one `ReentrantLock` with a single `notEmpty` `Condition`; array growth uses a separate CAS spin-lock so resizing doesn't block takers. | **Unbounded — `put` never blocks and `offer` always returns `true`.** No backpressure whatsoever. Iteration is not in priority order. Rejects `null`; elements must be `Comparable` or you must supply a `Comparator`. |
| **`SynchronousQueue`** | Yes (blocking, lock-free dual stack/queue) | • Put/Take: $O(1)$ (blocks until a counterparty arrives) | Direct hand-off with **maximum backpressure** — the producer cannot run ahead of the consumer at all. | Capacity **zero**; it holds no elements. Uses the Scherer–Lea dual stack (unfair) or dual queue (fair) to pair a waiting producer with a waiting consumer. | `size()` is always 0, `isEmpty()` always `true`, `peek()` always `null`, iteration always empty. Backs `Executors.newCachedThreadPool()` — which is why that pool spawns **unbounded threads**. `offer()` without a waiting consumer fails immediately. |
| **`DelayQueue`** | Yes (blocking, single lock) | • Offer: $O(\log N)$<br>• Take: $O(\log N)$ | Scheduled retry with backoff, TTL expiry, session timeouts, Kafka DLQ replay windows. | Unbounded `PriorityQueue` of `Delayed` elements under a `ReentrantLock`. An element is takeable only once `getDelay() <= 0`. Uses a **leader thread** (Leader–Follower pattern) so only one consumer timed-waits, avoiding a thundering herd. | Unbounded → no backpressure. `poll()` returns `null` when the head hasn't expired **even though the queue is non-empty**. `size()` counts expired *and* unexpired elements. |
| **`LinkedTransferQueue`** | Yes (lock-free CAS) | • Offer/Poll: $O(1)$<br>• `size()`: $O(N)$ | Message passing where the producer sometimes needs to know the item was actually consumed. | Dual queue holding either data or waiting-consumer nodes. `transfer(e)` blocks until a consumer receives `e`; `put(e)` behaves like a normal unbounded queue. | Unbounded — `put` never applies backpressure; only `transfer` does. `size()` is $O(N)$. Superset of `SynchronousQueue` and `ConcurrentLinkedQueue` behavior. |

---

## 4. Map Implementations

| Collection / Pattern | Thread-Safety | Complexity | Best Use Case / Workload | Internal Mechanics & Memory Behavior | Trade-offs & Anti-Patterns |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`HashMap`** | No | • Get/Put/Remove: Average $O(1)$, Worst $O(\log N)$<br>• `containsValue`: $O(N)$<br>• Iteration: $O(capacity + N)$ | Single-threaded default for key→value lookup when iteration order is irrelevant. **Default choice unless proven otherwise.** | `Node<K,V>[]` table; bucket index is `hash & (n-1)`. Hash is spread by `h ^ (h >>> 16)` to fold high bits down, since only the low bits survive the mask. Table allocated lazily on first `put` (capacity 16, load factor 0.75). Resize **doubles** capacity; because capacity is a power of two, an entry either stays at index `i` or moves to `i + oldCap`. Bins treeify at **≥8 in a bin AND ≥64 capacity**; untreeify at ≤6. | Not thread-safe; fail-fast iterators. $O(\log N)$ worst case holds **only if keys implement `Comparable`**. **Iteration cost is proportional to table capacity, not entry count** — a map that grew to 1M then shrank still iterates a huge sparse table. **Never mutate a field used in `hashCode()` after insertion** — the entry becomes unreachable but still occupies memory. Sizing on 17: `new HashMap<>((int)(n / 0.75f) + 1)`. |
| **`LinkedHashMap`** | No | • Get/Put/Remove: Average $O(1)$<br>• Iteration: $O(N)$ | Deterministic iteration order — reproducible output, ordered serialization, stable test fixtures. With `accessOrder=true`, the canonical **LRU cache**. | Extends `HashMap`; `Entry` adds `before`/`after` pointers forming a doubly-linked list across all entries. Iteration walks the link chain, so it is $O(N)$ **independent of capacity** — strictly better than `HashMap` for sparse maps. The 3-arg constructor `(capacity, loadFactor, accessOrder)` moves an entry to the tail on every `get`/`put`. | Two extra references per entry. **In access-order mode, `get()` is a structural modification** — it bumps `modCount`, so a concurrent read during iteration throws CME. Not thread-safe: an LRU built this way needs external synchronization. |
| **`TreeMap`** | No | • Get/Put/Remove: $O(\log N)$<br>• `firstKey`/`lastKey`: $O(\log N)$<br>• `containsValue`: $O(N)$ | Sorted iteration and **range queries** — the actual reason to choose it: `subMap`, `headMap`, `tailMap`, `floorKey`, `ceilingKey`, `higherKey`, `lowerKey`, `descendingMap`. | Red-black tree implementing `NavigableMap`. Ordering from `Comparable` or a supplied `Comparator`. Range views are **live windows** onto the same tree — writes through them mutate the backing map, and out-of-range writes throw `IllegalArgumentException`. | **Key uniqueness is defined by `compare() == 0`, not `equals()`.** An inconsistent comparator silently collapses distinct keys. Rejects `null` keys under natural ordering (allows `null` values). $O(\log N)$ everywhere, with poor cache locality vs. `HashMap`. |
| **`EnumMap`** | No | • Get/Put/Remove: $O(1)$, effectively an array index | Any map keyed by enum constants: per-state handlers, per-status counters, config-by-environment. **Always prefer over `HashMap<SomeEnum, V>`.** | Backed by a plain `Object[]` indexed directly by `Enum.ordinal()` — no hashing, no collisions, no `Node` objects. Requires the `Class<K>` token at construction. Dramatically smaller and faster than the hash equivalent. | Not thread-safe. Rejects `null` keys (`NPE`); `null` values allowed and distinguished from "absent" by a sentinel. Iteration is in **enum declaration order**, which cannot be changed. |
| **`WeakHashMap`** | No | • Get/Put/Remove: Average $O(1)$ | Attaching metadata to objects you don't own, keyed by identity lifetime — canonicalizing caches, per-`Class` or per-`ClassLoader` registries. | Entries extend `WeakReference` on the **key**. When a key becomes only weakly reachable, GC clears it and enqueues the entry on an internal `ReferenceQueue`; `expungeStaleEntries()` drains that queue during most map operations. | **Values are held strongly.** If a value transitively references its own key, the key is never collectable — a permanent leak. Fix by wrapping the value in a `WeakReference`. **`size()` can shrink with no modification by your code**, so it is unusable for exact bookkeeping. Keys held strongly elsewhere (interned `String`s, small boxed `Integer`s, enum constants) are never collected. **Not a cache** — no size bound, no TTL. Not thread-safe. |
| **`ConcurrentHashMap`** | Yes (CAS + per-bin `synchronized`) | • Get: Average $O(1)$, **lock-free**<br>• Put/Remove: Average $O(1)$<br>• `size()`: $O(1)$ estimate | **Default concurrent map.** Shared caches, session/registry state, dedup and idempotency tracking, rate-limiter buckets, in-flight request maps. | Java 8+ **abandoned the Java 7 `Segment[]` design entirely.** Same `Node[]` table as `HashMap`; reads are volatile array reads with no locking; installing the first node in an empty bin is a lock-free **CAS**; contended bins `synchronize` on the **head node only**, so unrelated bins never contend. Counters are striped (`baseCount` + `CounterCell[]`, LongAdder-style) to avoid a hot cache line. Resize is **cooperative** — a thread hitting a `ForwardingNode` helps transfer bins instead of blocking. | **Rejects `null` keys and values** — deliberately: `get()` returning `null` must unambiguously mean "absent", because you cannot re-check under a lock as you can with `HashMap`. `size()` is an **estimate**; use `mappingCount()` for a `long`. Weakly consistent iterators. **`computeIfAbsent`'s mapping function runs while holding the bin lock** — if it touches the same map you get deadlock or livelock. Keep those functions short, pure, non-blocking. Sequences of separate calls are still not atomic — use `compute`/`merge`/`putIfAbsent`. |
| **`ConcurrentSkipListMap`** | Yes (lock-free CAS) | • Get/Put/Remove: $O(\log N)$<br>• `size()`: $O(N)$ | Concurrent **sorted** map and concurrent range queries — the thread-safe `TreeMap`. Time-series buckets, ordered leaderboards, priority indexes. | Probabilistic multi-level skip list; nodes are promoted to higher index levels with ~1/2 probability. All updates are atomic pointer CAS — **no locks anywhere**. Implements `ConcurrentNavigableMap`, so `subMap`/`headMap`/`descendingMap` views are themselves concurrent and live. | Rejects `null` keys and values. **`size()` traverses the entire structure and is neither $O(1)$ nor atomic.** Higher per-entry memory than `TreeMap` (index nodes at multiple levels). Slower than `ConcurrentHashMap` for pure lookup — only pay the $O(\log N)$ if you actually need ordering. |
| **`Map.of` / `Map.ofEntries` / `Map.copyOf`** | Yes (immutable) | • Get: Average $O(1)$ | Constants, lookup tables, defensive returns from APIs, DTO fields. | `ImmutableCollections.MapN` — open-addressed probe table in a single flat array, no per-entry `Node` objects, lower footprint than `HashMap`. `Map.of` supports up to 10 pairs; beyond that use `Map.ofEntries(Map.entry(k, v), ...)`. | Rejects `null` keys and values. Throws `IllegalArgumentException` on **duplicate keys** at construction. **Iteration order is randomized per JVM run** via an internal `SALT`. All mutators throw `UnsupportedOperationException`. |
| **`Collections.unmodifiableMap`** | Depends on backing map | • Same as backing map | Exposing a read-only window onto internal state without copying. | A delegating **live view**. Not a copy. | **The backing map can still change underneath the view** — callers observe mutations they cannot make. `Map.copyOf` takes a true defensive copy; this does not. Neither makes the keys or values immutable — immutability is shallow. |
| **`Collections.synchronizedMap`** | Yes (single coarse mutex) | • Same as backing map + uncontended lock overhead | Legacy interop; retrofitting safety onto `LinkedHashMap` (e.g. a synchronized LRU), which has no concurrent equivalent. | Delegating wrapper; every method body is `synchronized (mutex)` on one monitor. **Java 8+ correctly overrides the default methods** — `putIfAbsent`, `compute`, `computeIfAbsent`, `merge`, `getOrDefault`, `forEach`, `replaceAll` are all synchronized and therefore genuinely atomic. | The problem is **granularity, not atomicity**: one lock serializes every reader and writer across the whole map. **Iteration is NOT protected** — you must manually `synchronized (map) { for (...) }` on the wrapper reference. A long-running `compute` blocks the entire map, whereas `ConcurrentHashMap` blocks only one bin. |

---

## 5. Cross-Cutting Cheat Sheets

### 5.1 Iterator Semantics — know which one you have

| Semantics | Collections | Behavior |
| :--- | :--- | :--- |
| **Fail-fast** | `ArrayList`, `LinkedList`, `Vector`, `HashSet`, `LinkedHashSet`, `TreeSet`, `ArrayDeque`, `PriorityQueue`, `HashMap`, `LinkedHashMap`, `TreeMap`, `EnumMap`, `WeakHashMap`, `IdentityHashMap`, all `Collections.synchronizedX` | Throws `ConcurrentModificationException` on structural modification during iteration (best-effort, via `modCount` — never depend on it). Only `iterator.remove()` is legal mid-loop; prefer `removeIf`. |
| **Snapshot** | `CopyOnWriteArrayList`, `CopyOnWriteArraySet` | Iterates a frozen array. Never throws CME, never sees writes made after creation. `remove`/`set`/`add` throw `UnsupportedOperationException`. |
| **Weakly consistent** | `ConcurrentHashMap` (+`newKeySet`), `ConcurrentSkipListMap`/`Set`, `ConcurrentLinkedQueue`/`Deque`, all `BlockingQueue`s | Never throws CME. Reflects state at creation and *may or may not* reflect later writes. Each element returned at most once. |

"Fail-safe" is blog vocabulary, not a JDK term, and it wrongly collapses the last two rows. Use the real words.

### 5.2 Null Handling

| Accepts `null` | Rejects `null` (`NullPointerException`) |
| :--- | :--- |
| `ArrayList`, `LinkedList`, `Vector`, `HashSet`, `LinkedHashSet`, `Arrays.asList`, `HashMap`(1 key), `LinkedHashMap`(1 key), `WeakHashMap`, `IdentityHashMap`, `EnumMap`(values only) | `ArrayDeque`, `PriorityQueue`, all `BlockingQueue`s, `ConcurrentLinkedQueue`/`Deque`, `ConcurrentHashMap`, `ConcurrentSkipListMap`/`Set`, `newKeySet`, `Hashtable`, `EnumSet`, `EnumMap`(keys), `List.of`/`Set.of`/`Map.of` |

`TreeMap`/`TreeSet` reject `null` under natural ordering but accept it if the supplied `Comparator` handles it.

**Why concurrent maps ban null:** with `HashMap` you disambiguate "mapped to null" from "absent" via `containsKey`. In a concurrent map that two-call sequence is racy and unfixable, so the ambiguity is banned at the API level instead.

### 5.3 `size()` Cost — the trap

| $O(1)$ and exact | $O(1)$ but an **estimate** | $O(N)$ traversal | Can change on its own |
| :--- | :--- | :--- | :--- |
| `ArrayList`, `ArrayDeque`, `PriorityQueue`, `HashSet`, `TreeSet`, `HashMap`, `LinkedHashMap`, `TreeMap`, `EnumMap`, `Hashtable`, `ArrayBlockingQueue`, `LinkedBlockingQueue` | `ConcurrentHashMap`, `ConcurrentHashMap.newKeySet()` | `ConcurrentLinkedQueue`/`Deque`, `ConcurrentSkipListSet`/`Map`, `LinkedTransferQueue` | `WeakHashMap` (GC clears keys) |

### 5.4 `HashMap` Internal Constants — memorize these

| Constant | Value | Meaning |
| :--- | :--- | :--- |
| `DEFAULT_INITIAL_CAPACITY` | 16 | Table size on first `put` (allocated lazily) |
| `DEFAULT_LOAD_FACTOR` | 0.75 | Resize when `size > capacity × 0.75` — the empirical balance between collision rate and wasted space |
| `MAXIMUM_CAPACITY` | $2^{30}$ | Hard ceiling |
| `TREEIFY_THRESHOLD` | 8 | Bin(bucket) length at which a chain converts to a red-black tree |
| `UNTREEIFY_THRESHOLD` | 6 | Bin(bucket) length at which a tree reverts to a chain (hysteresis prevents thrashing) |
| `MIN_TREEIFY_CAPACITY` | 64 | Below this capacity, a long bin triggers a **resize instead of treeification** |

**War story:** Java 7 `HashMap` used *head* insertion when transferring entries during resize, which under concurrent access could form a circular linked list and spin `get()` forever at 100% CPU. Java 8 switched to *tail* insertion, removing the infinite loop — but `HashMap` is still not thread-safe, and concurrent use still loses updates.

### 5.5 Java 8 `Map` Default Methods — write modern Java

| Method | Idiom |
| :--- | :--- |
| `getOrDefault(k, d)` | Replaces `containsKey` + `get` |
| `putIfAbsent(k, v)` | Insert only if absent; returns existing value or `null` |
| `computeIfAbsent(k, f)` | **Multimap idiom:** `map.computeIfAbsent(k, x -> new ArrayList<>()).add(v)` |
| `computeIfPresent(k, f)` | Update only if present; returning `null` removes the entry |
| `compute(k, f)` | Read-modify-write in one atomic step (on `ConcurrentHashMap`) |
| `merge(k, v, f)` | **Counting idiom:** `map.merge(word, 1, Integer::sum)` |
| `forEach(BiConsumer)` | Cleaner than `entrySet()` iteration |
| `replaceAll(BiFunction)` | In-place bulk transform |

Atomic on `ConcurrentHashMap`; merely convenient on `HashMap`. The `computeIfAbsent` bin-lock caveat in §4 applies.

### 5.6 `BlockingQueue` — the four operation flavors

| | Throws exception | Returns special value | Blocks | Times out |
| :--- | :--- | :--- | :--- | :--- |
| **Insert** | `add(e)` | `offer(e)` | `put(e)` | `offer(e, t, u)` |
| **Remove** | `remove()` | `poll()` | `take()` | `poll(t, u)` |
| **Examine** | `element()` | `peek()` | — | — |

### 5.7 `ThreadPoolExecutor` Queue Interaction — high-yield interview material

Growth rule: **fill core threads → fill the queue → only then grow toward `maximumPoolSize` → then apply the `RejectedExecutionHandler`.**

| Queue choice | Consequence |
| :--- | :--- |
| Unbounded `LinkedBlockingQueue` (`newFixedThreadPool`) | Queue never fills → pool never exceeds `corePoolSize` → `maximumPoolSize` is ignored → unbounded memory growth → OOM. |
| `SynchronousQueue` (`newCachedThreadPool`) | Every task needs an immediate thread → pool grows to `maximumPoolSize` (`Integer.MAX_VALUE`) → thread explosion. |
| Bounded `ArrayBlockingQueue` | Correct production shape: bounded memory, real backpressure, `maximumPoolSize` actually reachable, explicit rejection policy. |

### 5.8 LRU Cache Recipe

```java
// Bounded, access-ordered LRU in ~5 lines.
Map<K, V> lru = new LinkedHashMap<>(capacity, 0.75f, true) {   // true = access order
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
};
// Not thread-safe. Wrap with Collections.synchronizedMap, or use Caffeine in production.
```

Follow-ups to expect: why `accessOrder` makes `get()` a structural modification; how you'd do this **without** `LinkedHashMap` (`HashMap` + your own doubly-linked list, $O(1)$ per op); and why production should use Caffeine (TTL, weight-based eviction, async refresh, W-TinyLFU admission).

### 5.9 Every `Set` Is a `Map` in Disguise

| Set | Backing map |
| :--- | :--- |
| `HashSet` | `HashMap` with a shared `PRESENT` dummy value |
| `LinkedHashSet` | `LinkedHashMap` |
| `TreeSet` | `TreeMap` |
| `ConcurrentSkipListSet` | `ConcurrentSkipListMap` |
| `ConcurrentHashMap.newKeySet()` | `ConcurrentHashMap` with `Boolean.TRUE` |
| `CopyOnWriteArraySet` | `CopyOnWriteArrayList` (not a map) |
| `EnumSet` | *(exception — bit vector, not map-backed)* |

### 5.10 Selection Heuristics — the one-page answer

| Requirement | Single-threaded | Concurrent |
| :--- | :--- | :--- |
| Ordered list, index access | `ArrayList` | `CopyOnWriteArrayList` (read-heavy only) |
| Unique membership | `HashSet` | `ConcurrentHashMap.newKeySet()` |
| Unique + insertion order | `LinkedHashSet` | *(none built-in)* |
| Unique + sorted / range | `TreeSet` | `ConcurrentSkipListSet` |
| Set of enum constants | `EnumSet` | `EnumSet` + external sync |
| Key→value lookup | `HashMap` | `ConcurrentHashMap` |
| Map + predictable order | `LinkedHashMap` | `synchronizedMap(new LinkedHashMap<>())` |
| Map + sorted / range | `TreeMap` | `ConcurrentSkipListMap` |
| Map keyed by enum | `EnumMap` | `EnumMap` + external sync |
| Bounded LRU cache | `LinkedHashMap` (access-order) | Caffeine / Guava — not the JDK |
| Metadata by object lifetime | `WeakHashMap` | `synchronizedMap(new WeakHashMap<>())` |
| Reference-identity keys | `IdentityHashMap` | — |
| FIFO queue / LIFO stack | `ArrayDeque` | `ConcurrentLinkedQueue` (non-blocking) or `ArrayBlockingQueue` (backpressure) |
| Priority ordering | `PriorityQueue` | `PriorityBlockingQueue` |
| Producer–consumer + backpressure | — | Bounded `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| Direct hand-off, zero buffering | — | `SynchronousQueue` |
| Time-delayed release | — | `DelayQueue` |
| Constants / defensive returns | `List.of` / `Set.of` / `Map.of` | Same (immutable is inherently safe) |

---

## Appendix — Deliberately Excluded from the Tables

These are **not production choices**, so they are kept out of §1–§4 to preserve the tables' single purpose: *what do I reach for?* They remain interview material.

### Legacy — know them, never write them

| Class | Replaced by | The one fact interviewers want |
| :--- | :--- | :--- |
| `Vector` | `ArrayList` (+ explicit sync) | Synchronized `ArrayList`; grows $2\times$ instead of $1.5\times$. |
| `Stack` | `ArrayDeque` | Extends `Vector`, so it inherits the coarse lock. **Iterates bottom-to-top** — the opposite of `ArrayDeque`-as-a-stack. A real migration bug. |
| `Hashtable` | `ConcurrentHashMap` | **Rejects `null` keys AND values**, whereas `HashMap` allows one null key and many null values. The most-asked `HashMap` vs `Hashtable` difference. Capacity 11, resize `2n+1` (modulo, not masking). |
| `Enumeration` | `Iterator` | Legacy traversal, not fail-fast. `Hashtable` exposes both. |

### Special-purpose — legitimate, but you'd know if you needed one

| Class | When it's actually correct |
| :--- | :--- |
| `IdentityHashMap` | Object-graph traversal where two `equals` objects must stay distinct: cycle detection, deep-copy and serialization bookkeeping. Compares with `==` and `System.identityHashCode()`; stores keys and values in one flat `Object[]` with linear probing. **Deliberately violates the `Map` contract** — never a general-purpose map. |
| `Properties` | `.properties` files and `System.getProperties()`. Extends `Hashtable<Object,Object>` with broken generics — use `getProperty`/`setProperty`, never the inherited `get`/`put`. |
| `LinkedList` | Retained in §1 only as the reference point for why `ArrayList` and `ArrayDeque` win. No workload selects it. |

### Not in the JDK, but the right production answer

| Need | Use |
| :--- | :--- |
| Cache with TTL, size/weight bounds, async refresh, stats | **Caffeine** (or Guava `Cache`). `LinkedHashMap`-LRU is a whiteboard answer, not a production one. |
| Multimap, BiMap, immutable collections with ordering guarantees | **Guava** |
| Primitive collections without boxing (`int → long` at scale) | **Eclipse Collections**, **fastutil**, **HPPC** — a `HashMap<Integer, Long>` costs ~50 bytes per entry versus ~16. |
