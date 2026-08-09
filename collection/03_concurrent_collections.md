# Concurrent Collections

> Baseline: **Java 17 (LTS)**. Covers `java.util.concurrent` + the ways to make any collection thread-safe.

---

## 0. The Thread-Safety Spectrum

| Level | Example | What you actually get |
| :--- | :--- | :--- |
| **Unsafe** | `ArrayList`, `HashMap` | Nothing. Fail-fast iterator is a *bug detector*, not a safety mechanism. |
| **Coarse-locked wrapper** | `Collections.synchronizedList/Map` | Every method atomic, but **one lock for the whole collection**. Iteration NOT protected. |
| **Legacy synchronized class** | `Vector`, `Hashtable`, `Stack` | Same as above, baked into the class. Never write new code with these. |
| **Immutable** | `List.of()`, `Map.copyOf()` | Thread-safe by having no writes at all. Shallow — elements can still be mutable. |
| **Concurrent** | `ConcurrentHashMap`, `ArrayBlockingQueue` | Fine-grained locking or lock-free CAS. Multiple threads make progress simultaneously. |

**The three rules that matter:**
1. Individual methods being atomic ≠ your **compound operation** is atomic. `if (!map.containsKey(k)) map.put(k, v)` is racy on *every* collection above. Use `putIfAbsent` / `compute` / `merge`.
2. Thread-safety of the collection says **nothing** about the objects inside it. A `ConcurrentHashMap<String, MutableOrder>` gives you zero protection on `MutableOrder`.
3. The real question is never "is it thread-safe?" — it is **"where is the contention, and is it bounded?"**

---

## 1. List

### `CopyOnWriteArrayList`

1. Only concurrent `List` in the JDK. Mutation creates a **whole new array**; the old one is never touched.
2. Writes: `synchronized` on an internal lock object (JDK 9+; was `ReentrantLock` earlier) → `Arrays.copyOf` → publish via `volatile` field swap.
3. Reads: read the `volatile Object[]` and index into it. **Zero synchronization, zero blocking, `get()` is O(1).**
4. Iterator is a **snapshot** — frozen at creation. Never throws `CME`, never sees later writes.
5. `iterator.remove()` / `set()` / `add()` throw `UnsupportedOperationException` — the snapshot is not the live array. Use `removeIf()` on the list itself.
6. Allows `null`. `addIfAbsent()` / `addAllAbsent()` are the extra methods.
7. Every write is **O(N) time + O(N) allocation** → GC pressure. Two sequential `add()` calls = two full copies.
8. Rule of thumb: **>95% reads, small N (< a few thousand)**.
9. Use case: listener registries, routing tables, feature flags, cached config read on every request and updated on deploy.
10. Anti-pattern: any write-heavy path, or a large list. This is the single most misused concurrent collection.

---

## 2. Set

| | `CopyOnWriteArraySet` | `ConcurrentSkipListSet` | `ConcurrentHashMap.newKeySet()` |
| :--- | :--- | :--- | :--- |
| **Backed by** | `CopyOnWriteArrayList` | `ConcurrentSkipListMap` | `ConcurrentHashMap` (value = `Boolean.TRUE`) |
| **Uniqueness by** | linear `equals()` scan (`addIfAbsent`) | `compare() == 0` | `hashCode()` + `equals()` |
| **Ordering** | insertion | **sorted** (`NavigableSet`) | none |
| **Add / Remove / Contains** | O(N) scan + O(N) copy | O(log N) | avg O(1) |
| **Mechanism** | copy-on-write, lock-free reads | lock-free CAS on pointers | CAS on empty bin, `synchronized` on head node |
| **`size()`** | O(1) | **O(N) traversal**, not atomic | O(1) **estimate** |
| **Iterator** | snapshot | weakly consistent | weakly consistent |
| **`null`** | allowed | rejected | rejected |
| **Use case** | tiny (N<100) read-only-mostly registries | concurrent sorted set / range queries — the thread-safe `TreeSet` | **default concurrent set** — dedup, idempotency keys, in-flight request tracking |

**Notes:**
1. `newKeySet()` is the answer to "concurrent `HashSet`" — there is no `ConcurrentHashSet` class.
2. `CopyOnWriteArraySet` pays a **double penalty**: `contains()` is O(N) even for pure reads (no hashing path at all), unlike `CopyOnWriteArrayList` which at least has O(1) `get(i)`.
3. **There is no concurrent insertion-ordered set** (no concurrent `LinkedHashSet`). Fall back to `synchronizedSet(new LinkedHashSet<>())`.

---

## 3. Queue

Split on one axis first: **does an empty/full queue block the caller?**

### 3.1 Non-blocking (lock-free)

| | `ConcurrentLinkedQueue` | `ConcurrentLinkedDeque` |
| :--- | :--- | :--- |
| **Structure** | Michael–Scott singly-linked, CAS on `volatile` node pointers, lazy head/tail advance | doubly-linked, CAS both ends |
| **Complexity** | offer/poll O(1) amortized | O(1) at either end |
| **Bounded?** | **Unbounded** | **Unbounded** |
| **`size()`** | **O(N)** walk, estimate only — use `isEmpty()` | **O(N)** |
| **Use case** | high-throughput handoff where a consumer must never block — work stealing, metrics buffers, actor mailboxes | same, when you need both ends |
| **Risk** | no backpressure → slow consumer = OOM | same |

### 3.2 Blocking

| | Bounded? | Lock structure | `size()` | Key production fact |
| :--- | :--- | :--- | :--- | :--- |
| **`ArrayBlockingQueue`** | **Always** (fixed at construction) | **one** `ReentrantLock` + `notEmpty`/`notFull` Conditions | O(1) | Pre-allocated ring buffer, **zero node allocation**. Producers and consumers contend on the same lock. The safe default for a `ThreadPoolExecutor`. |
| **`LinkedBlockingQueue`** | Only if capacity passed | **two** locks (`putLock`, `takeLock`) + `AtomicInteger` count | O(1) | Producer and consumer run in parallel → higher throughput. **Defaults to `Integer.MAX_VALUE`** — this is the OOM in `newFixedThreadPool`. |
| **`LinkedBlockingDeque`** | Optional | **one** lock (both ends can meet) | O(1) | Loses the dual-lock advantage. Also defaults to `Integer.MAX_VALUE`. Work-stealing, retry queues (push failures back to head). |
| **`PriorityBlockingQueue`** | **Unbounded** | one lock + `notEmpty` only; separate CAS spinlock for array growth | O(1) | `put` never blocks, `offer` always returns `true` → **no backpressure at all**. Iteration not in priority order. |
| **`DelayQueue`** | **Unbounded** | one lock, **leader–follower** (only one consumer timed-waits) | O(1) | Elements implement `Delayed`. `poll()` returns `null` while head hasn't expired **even though queue is non-empty**. Retry-with-backoff, TTL expiry, DLQ replay. |
| **`SynchronousQueue`** | **Capacity 0** | Scherer–Lea dual stack (unfair) / dual queue (fair) | always 0 | Holds nothing. Pure hand-off = **maximum backpressure**. Backs `newCachedThreadPool` → unbounded thread growth. |
| **`LinkedTransferQueue`** | **Unbounded** | lock-free CAS dual queue | **O(N)** | `transfer(e)` blocks until a consumer actually takes `e`; `put(e)` doesn't. Superset of `SynchronousQueue` + `ConcurrentLinkedQueue`. |

### 3.3 The four operation flavours (`BlockingQueue`)

| | Throws | Returns special value | Blocks | Times out |
| :--- | :--- | :--- | :--- | :--- |
| **Insert** | `add(e)` | `offer(e)` | `put(e)` | `offer(e, t, u)` |
| **Remove** | `remove()` | `poll()` | `take()` | `poll(t, u)` |
| **Examine** | `element()` | `peek()` | — | — |

### 3.4 Queue rules

1. **Bounded vs unbounded is the whole game.** Unbounded = no backpressure = the queue absorbs the failure until the JVM dies. Always pass an explicit capacity.
2. All concurrent/blocking queues **reject `null`** — `null` is the sentinel for "empty".
3. All of them have **weakly consistent** iterators. **No queue offers snapshot iteration.**
4. `size()` is O(N) on `ConcurrentLinkedQueue`/`Deque` and `LinkedTransferQueue`; O(1) on the array/linked blocking queues.
5. `ThreadPoolExecutor` growth order: **fill core threads → fill the queue → grow toward max → then reject.** An unbounded queue means step 3 never happens and `maximumPoolSize` is dead configuration.

---

## 4. Map

### 4.1 `ConcurrentHashMap` — the default concurrent map

1. **Java 8 abandoned the Java 7 `Segment[]` design entirely.** Do not describe segments/stripe-16 in an interview — that answer dates you.
2. Same `Node<K,V>[]` table as `HashMap`, same hash spread `h ^ (h >>> 16)`, same treeify thresholds (≥8 in bin AND ≥64 capacity).
3. **Reads are lock-free** — `volatile` array read + `volatile` node fields. `get()` never blocks, ever.
4. **Writes:** empty bin → lock-free **CAS** to install the first node. Contended bin → `synchronized` on the **head node only**, so unrelated bins never contend. Lock granularity = one bucket.
5. **Rejects `null` keys AND values — deliberately.** With `HashMap` you disambiguate "mapped to null" from "absent" via `containsKey`. Concurrently that two-call sequence is racy and unfixable, so the ambiguity is banned at the API level.
6. `size()` is an **estimate** from striped counters (`baseCount` + `CounterCell[]`, LongAdder-style, to avoid one hot cache line). Use `mappingCount()` for a `long`. Never use either for exact invariants.
7. **Resize is cooperative**: a thread that hits a `ForwardingNode` helps transfer bins instead of blocking.
8. Iterators are **weakly consistent** — never throw `CME`, may or may not reflect writes after creation, each element returned at most once.
9. Atomic compound ops: `putIfAbsent`, `compute`, `computeIfAbsent`, `computeIfPresent`, `merge`. These are the *only* atomic way to do read-modify-write. A sequence of separate calls is still racy.
10. **`computeIfAbsent`'s mapping function runs while holding the bin lock.** If it touches the same map you get deadlock, livelock, or `IllegalStateException: Recursive update`. Keep those functions **short, pure, non-blocking** — never an I/O or DB call.
11. Bulk parallel ops with a parallelism threshold: `forEach`, `search`, `reduce` (+`reduceKeys`/`reduceValues`). Rarely needed; know they exist.
12. Use case: shared caches, session/registry state, dedup and idempotency tracking, rate-limiter buckets, in-flight request maps.

### 4.2 `ConcurrentSkipListMap`

1. The thread-safe `TreeMap`. Probabilistic multi-level linked list; nodes promoted to higher index levels with ~1/2 probability.
2. **All updates are atomic pointer CAS — no locks anywhere.**
3. O(log N) get/put/remove. Rejects `null` keys and values.
4. Implements `ConcurrentNavigableMap` — `subMap` / `headMap` / `tailMap` / `descendingMap` views are themselves concurrent and **live**.
5. **`size()` traverses the entire structure — O(N) and not atomic.**
6. Higher per-entry memory than `TreeMap` (index nodes at multiple levels). Slower than `ConcurrentHashMap` for pure lookup — only pay O(log N) if you actually need ordering.
7. Use case: time-series buckets, ordered leaderboards, priority indexes, concurrent range queries.

### 4.3 Map comparison

| | `Hashtable` | `Collections.synchronizedMap` | `ConcurrentHashMap` | `ConcurrentSkipListMap` |
| :--- | :--- | :--- | :--- | :--- |
| **Lock granularity** | whole object | whole object (one mutex) | **one bin** | **none** (CAS) |
| **Reads block?** | Yes | Yes | **No** | No |
| **Ordering** | none | that of the backing map | none | **sorted** |
| **`null` key/value** | rejects both | depends on backing map | rejects both | rejects both |
| **Iterator** | fail-fast (`Iterator`) / `Enumeration` not fail-fast | fail-fast, **not protected** — you must sync manually | weakly consistent | weakly consistent |
| **Status** | legacy — never write it | legacy interop only | **default** | default when sorted |

---

## 5. Other Ways to Get a Thread-Safe Collection

### 5.1 `Collections.synchronizedX(...)`
* `synchronizedList` / `Set` / `Map` / `SortedMap` / `NavigableMap` / `SortedSet` / `Collection`.
* Delegating wrapper; every method body is `synchronized (mutex) { ... }` on **one** monitor.
* Java 8+ correctly overrides the default methods (`putIfAbsent`, `compute`, `merge`, `getOrDefault`, `forEach`, `replaceAll`) — so they *are* genuinely atomic. The problem is **granularity, not atomicity**.
* **Iteration is NOT protected.** You must do it yourself:
```java
synchronized (syncList) {          // lock on the WRAPPER reference, not the backing list
    for (T t : syncList) { ... }
}
```
* **Legitimate remaining use:** wrapping a `LinkedHashMap` (LRU / insertion-ordered map) or `WeakHashMap`, which have no concurrent equivalent. For everything else, `ConcurrentHashMap` wins.

### 5.2 Immutable collections
* `List.of` / `Set.of` / `Map.of` / `Map.ofEntries`, `List.copyOf` etc., `Stream.toList()`.
* Thread-safe because there are no writes. **No locks, no copying, no contention — the cheapest thread-safety there is.**
* **Shallow:** the *elements* can still be mutable and unsafely shared.
* `Collections.unmodifiableX` is **NOT** thread-safe — it's a live view over a mutable backing collection that another thread can be writing to.
* Pattern: hold an **immutable snapshot in an `AtomicReference`** and swap it on update. This is hand-rolled copy-on-write with explicit control over the copy cost.

### 5.3 Confinement
* **`ThreadLocal`** — no sharing, no synchronization needed. Must `remove()` in a `finally` on pooled threads or you leak (and leak across requests).
* Stack confinement (local variables), or single-threaded ownership with a queue as the only entry point (actor model).

### 5.4 External locking
* `ReentrantLock` (+ `Condition`) — `tryLock` with timeout, interruptible, fairness option.
* `ReentrantReadWriteLock` — many readers or one writer. Beware writer starvation in unfair mode.
* `StampedLock` — adds **optimistic reads** (`tryOptimisticRead` + `validate`). Not reentrant. Fastest for read-dominated data with short critical sections.
* Use when the invariant spans **multiple collections** — no single concurrent collection can protect that.

### 5.5 Atomics
* `AtomicInteger` / `AtomicLong` / `AtomicReference` — CAS on a single variable.
* `LongAdder` / `LongAccumulator` — striped counters, far better than `AtomicLong` under **high write contention** (this is what `ConcurrentHashMap.size()` uses internally).

### 5.6 Not in the JDK
* **Caffeine** (or Guava `Cache`) for any real cache — TTL, size/weight bounds, async refresh, W-TinyLFU admission, stats. A `synchronizedMap(LinkedHashMap)` LRU is a whiteboard answer, not a production one.

---

## 6. Interview Traps

1. "Is `ConcurrentHashMap` fully thread-safe?" → Individual ops yes; **your compound sequence no**. And it doesn't protect the objects stored in it.
2. Describing `ConcurrentHashMap` with segments → Java 7 answer. Say **CAS on empty bin, `synchronized` on head node**.
3. Why does `ConcurrentHashMap` reject `null`? → to make `get() == null` unambiguously mean "absent"; the `containsKey` re-check is racy concurrently.
4. `size()` on `ConcurrentHashMap` is an **estimate**; on `ConcurrentSkipListMap` / `ConcurrentLinkedQueue` it's **O(N)**. Use `isEmpty()`.
5. Snapshot ≠ weakly consistent. COW freezes; concurrent views **may** show later writes. "Fail-safe" is blog vocabulary, not a JDK term.
6. `CopyOnWriteArrayList.iterator().remove()` → `UnsupportedOperationException`, not `CME`.
7. `newFixedThreadPool` OOMs because of the **unbounded `LinkedBlockingQueue`**, not the threads. `newCachedThreadPool` explodes threads because of `SynchronousQueue`.
8. `Collections.synchronizedMap` + iteration without an external `synchronized` block → `CME` under concurrency. Very common bug.
9. Long-running function inside `computeIfAbsent` on a `ConcurrentHashMap` → holds the bin lock; touching the same map → recursive update / deadlock.
10. There is no concurrent `LinkedHashSet` / `LinkedHashMap`. Use `synchronizedMap(new LinkedHashMap<>())` or Caffeine.
11. `Vector` / `Hashtable` are "thread-safe" and still wrong — coarse lock, and compound ops are racy anyway.
12. Every mutable-key rule from `HashMap` still applies: mutating a field used in `hashCode()` after insert makes the entry unreachable, concurrent or not.
