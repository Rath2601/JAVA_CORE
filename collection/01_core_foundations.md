## **Important Terms** :

* **Iterable**: Marks a collection that can be iterated; provides the iterator() method acting as a factory that returns a new Iterator instance.
* **Iterator**: Used to traverse a collection; acts as a single-use cursor offering methods like hasNext(), next(), and remove().

NOTE:
* **State Separation**:Each iterator() call returns a fresh cursor, nested loops don't corrupt each other. (**Fail-fast**) java throw CME, or may silently skip elements (if we remove second last element) if we mutate element while iterating either in single / multithreaded environment
* **Single-Use Lifecycle**: Once exhausted (hasNext() -> false), an Iterator cannot be reset. (ListIterator- > bidirectional, still can't reset)
* **Safe Mutation**: iterator.remove() avoids ConcurrentModificationException. The collection counts structural changes (add/remove/resize, not set()) in **modCount**; the iterator snapshots it as **expectedModCount** at creation, so mutating via the collection desyncs the two. remove() is optional (UOE on immutable collections) and valid only once per next()
* For Map (doesn't implement Iterable): use a view — entrySet(), keySet(), values() — or map.forEach(). each view is a collection which follows iterator rule

---
## Iterator Behaviour: Fail-Fast vs Non-Fail-Fast
| Aspect | Fail-Fast (for single-threaded use; detects mutation anywhere, but unreliably under concurrency) | Non-Fail-Fast (Snapshot / Weakly Consistent) (built for concurrency; works single-threaded too) |
|---|---|---|
| **Core idea** | Detects mutation during iteration and aborts by throwing `ConcurrentModificationException`. Best-effort bug detector, not a safety feature. | Never throws. Either iterates a frozen copy (**snapshot**) or the live structure with relaxed guarantees (**weakly consistent**). |
| **Collections** | `ArrayList`, `LinkedList`, `HashMap`, `HashSet`, `TreeMap`, `TreeSet`, `Vector`, and all `Collections.synchronizedXxx` wrappers (the wrapper delegates to the underlying fail-fast iterator). | **Snapshot — COW (copy-on-write):** `CopyOnWriteArrayList`, `CopyOnWriteArraySet`<br>**Weakly consistent:** `ConcurrentHashMap` (+ its `keySet`/`values`/`entrySet` views), `ConcurrentLinkedQueue/Deque`, `ConcurrentSkipListMap/Set`, `LinkedBlockingQueue`, `ArrayBlockingQueue` |
| **In-loop removal** | Safe: `Iterator.remove()`, `removeIf()`, `ListIterator.add()/set()`. These either update the iterator's `expectedModCount` to match `modCount`, or don't iterate at all.<br>Unsafe: any mutation via the collection itself (`list.remove(x)`, `map.put(k,v)`). | **COW:** iterator's `remove()`/`set()`/`add()` throw `UnsupportedOperationException` — it holds the old array, which is now immutable, so removing from it would change nothing. Mutate via the collection (`list.remove(x)`, `removeIf()`) instead.<br>**Weakly consistent:** `remove()` works normally — the iterator walks the live structure, so it deletes from the actual collection. |
| **Thread safety** | **Not thread-safe.** CME detects *any* mutation, single- or multi-threaded — it's a mutation detector, not a concurrency guard. Concurrent reads are safe only while nobody writes. | **Thread-safe.**<br>**COW:** each write copies the array and swaps the reference atomically under a lock; readers hold their own reference and never block.<br>**CHM:** writes lock only the affected bin; reads take no lock. Traversal returns each surviving element exactly once even across a resize — which is why it never needs to fail. |
| **Performance & memory** | Cheapest option — one `int` comparison per `next()`, no copying, no locking. | **COW:** reads are lock-free; every single write copies the entire array → O(n) per mutation, O(n²) to build a list of n elements. Both old and new arrays are live during the copy.<br>**CHM:** read speed close to `HashMap`; writers contend only when hitting the same bin. |
| **Risks** | Unexpected CME reaching production.<br>Silent element skips: `hasNext()` doesn't check `modCount`, so if a removal makes `cursor == size` the loop exits early with elements unvisited and no exception. | **COW:** iterator sees a snapshot frozen at creation — writes made after are invisible for the rest of the loop, so readers act on stale data. Plus GC pressure and memory spikes on large or write-heavy lists; use only when reads vastly outnumber writes.<br>**CHM:** concurrent inserts may or may not appear mid-traversal — undefined, never depend on it. `size()`, `isEmpty()`, `containsValue()` are estimates under mutation — never branch on them. `get()`-then-`put()` is not atomic; use `computeIfAbsent`, `merge`, or `putIfAbsent`. |
| **Use when** | Single-threaded code, or shared state already guarded by external synchronization. Treat a CME as a bug signal, not something to catch. | **COW:** shared list that is read constantly and mutated rarely (listener lists, config snapshots).<br>**CHM:** hot concurrent map — caches, counters, dedup registries. |

---

---
### **Marker Interface** :
It merely marks a class as capable of doing a specific task to the JRE.

1. **Serializable**  : marks a particular class can be serialized.
2. **Clonable**      : indicates a class permits cloning.
3. **Random access** : to mark list implementations that supports fast random access.

If your class does not implement Serializable (or extend Throwable), you do not need serialVersionUID. In modern Spring Boot, avoid implementing Serializable altogether unless forced by a legacy framework interface.

**serialVersionUID** : long value uniquely identifies a version of a serializable class.
* How to Generate -> Automatic Generation by IDE / Manual assignment
* Minor Changes -> If change is backward compatible leave as is it.
  Major Changes -> increment the serialVersionUID to signal a new version.

---
### **Object class methods** :
The hashCode() and equals() methods from the Object class are required by collections for the following operations:

1. Sequential Collections (List, Queue)
* Insertion (add): No methods needed. Elements are blindly appended sequentially.
* Deletion & Access (remove, contains, indexOf): Require only equals(). The collection performs a linear scan to find a match.
* hashCode(): Not required or used.

2. Hashing Collections (HashSet, HashMap, Hashtable)
* Insertion (put, add): Require BOTH hashCode() and equals(). hashCode() finds the bucket. If a collision occurs, equals() checks if the key exists to overwrite it; otherwise, it appends a new node.
* Deletion & Access (remove, get, containsKey): Require BOTH hashCode() and equals(). hashCode() instantly targets the bucket, and equals() pinpoints the exact object inside it.

3. Custom Class Rules
* The exact same principles above apply to custom classes when stored in these collections.
* when overriding equals() first to check (this == obj). If references match, it instantly returns true, skipping expensive field-by-field comparisons

---
### **Comparable & Comparator**

1. `Comparable` → implemented on the class itself (natural order). `Comparator` → external (separate class / lambda / method reference) for alternative ordering as per logic, as many as we want. (decoupled from the source class)
2. Methods: `Comparable` → `compareTo(T o)` (1 arg: `this` vs `o`). `Comparator` → `compare(T o1, T o2)` (2 args).
3. `Comparator` precedes over `Comparable`. (overrides natural ordering only when explicitly supplied in constructor or sort methods)
4. `Collections.sort()` / `List.sort()` uses TimSort (O(n log n)).`Arrays.sort()` — works on arrays; TimSort for reference-object arrays (`Integer[]`), Dual-Pivot Quicksort for primitive arrays (`int[]`).
5. `TreeSet` / `TreeMap` need `Comparable` or a `Comparator`, else `ClassCastException` at runtime; they order + de-dupe by comparison, ignoring `equals()` / `hashCode()`.
6. If domain class used across both sorted (compareTo/compare) and hash/equality-based collections (equals/hashCode), ensure (a.compareTo(b) == 0) must equal a.equals(b)

---

### Unmodifiable Collection
**Description:** Read-only view of collection. Changes to original collection will be reflected here, while mutation throws `UnsupportedOperationException`. Null elements allowed. Not thread-safe (reflects original's concurrent modifications).

#### Use Case
* Wrap third party / older code returning mutable list before passing to downstream
* Only owning class adds/removes items; other classes can't

#### Java Classes
* `Collections.unmodifiableList()` / `..Set()` / `..Map()` — **Java 1.2**
* `Collectors.toUnmodifiableList()` / `..Set()` / `..Map()` — **Java 10** *(used in Streams)*

---

### Immutable Collection
**Description:** Independent collection. Mutation throws `UnsupportedOperationException`. Null elements not allowed. Thread safe *(if contained elements are immutable)*.

#### Use Case
* `@Cacheable` returns fixed snapshot; no caller can corrupt it
* Bootstrap servers, group IDs set once at startup and never mutated again

#### Java Classes
* `List.of()` / `Set.of()` / `Map.of()` — **Java 9**
* `List.copyOf()` / `Set.copyOf()` / `Map.copyOf()` — **Java 10**
* `Stream.toList()` — **Java 16** *(only list; unmodifiable, allows nulls from stream)*

---
#### 1. Structurally Immutable vs. Truly Immutable
* **Structurally Immutable (e.g., `Arrays.asList()`):** Fixed-size list. Structural mutations (`add()`, `remove()`) throw `UnsupportedOperationException`, but **element replacement via `set(index, element)` is allowed**.
* **Truly Immutable Collection (e.g., `List.of()`, `List.copyOf()`):** Neither structural changes (`add()`, `remove()`) nor element replacements (`set()`) are allowed. Calling `set()` throws `UnsupportedOperationException`.

#### 2. Mutable Objects Inside a "Truly Immutable" Collection (Shallow Immutability)
* `List.of()` guarantees **shallow immutability** (the collection references cannot be added, removed, or replaced).
* **If elements inside are mutable:** You **can** manipulate internal object state (e.g., `list.get(0).setName("New")`). The collection reference remains untouched, but state changes inside the object reflect across all callers.
* **If elements inside are immutable:** Any attempt to update an immutable element (e.g., `String`, `Integer`, or a Java `record`) requires creating a new object, which must then be assigned to a **new list altogether** because `set()` is forbidden.

---
| Concurrency Mechanism | Blocking OS Threads? | Hardware Level Used | Primary Advantage | Major Trade-offs / Bottlenecks | Primary Real-World Use Case |
|---|---|---|---|---|---|
| **synchronized** | Yes | OS Mutex / Intrinsic Monitor | Simple, safe coarse-grained protection | Heavy thread context-switching overhead (~µs) | Legacy synchronization wrappers, CHM bucket lock |
| **CAS (Lock-Free)** | No | CPU CMPXCHG((Compare and Exchange) is an x86 instruction) instruction | Zero thread blocking; maximum throughput | Burns CPU in retry loops under high contention(போராட்டம்) | High-frequency counters, concurrent map insertion |
| **Copy-On-Write** | No (for readers) | `volatile` array pointers | Zero-lock $O(1)$ reads | High GC pressure & $O(N)$ array copy cost on writes | Config registries, routing tables, listener lists |
| **ReentrantLock / AQS (AbstractQueuedSynchronizer)** | Yes (when blocked) | AQS state + CAS queuing | Fine-grained lock control, split locks, timeouts | Thread parking/unparking when limits are hit | Bounded execution queues, rate limiters, payment pipelines |
