## **Important Terms** :

* **Iterable**: Marks a collection that can be iterated; provides the iterator() method acting as a factory that returns a new Iterator instance.
* **Iterator**: Used to traverse a collection; acts as a single-use cursor offering methods like hasNext(), next(), and remove().

NOTE:
* **State Separation**:Each iterator() call returns a fresh cursor, nested loops don't corrupt each other. (**Fail-fast**) java throw CME, or may silently skip elements (if we remove second last element) if we mutate element while iterating either in single / multithreaded environment
* **Single-Use Lifecycle**: Once exhausted (hasNext() -> false), an Iterator cannot be reset. (ListIterator- > bidirectional, still can't reset)
* **Safe Mutation**: iterator.remove() avoids ConcurrentModificationException. The collection counts structural changes (add/remove/resize, not set()) in **modCount**; the iterator snapshots it as **expectedModCount** at creation, so mutating via the collection desyncs the two. remove() is optional (UOE on immutable collections) and valid only once per next()
* For Map (doesn't implement Iterable): use a view — entrySet(), keySet(), values() — or map.forEach(). each view is a collection which follows iterator rule

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
4. `Collections.sort()` / `List.sort()` — work on `List` & uses TimSort (O(n log n)).
5. `Arrays.sort()` — works on arrays; TimSort for reference-object arrays (`Integer[]`), Dual-Pivot Quicksort for primitive arrays (`int[]`).
6. `TreeSet` / `TreeMap` need `Comparable` or a `Comparator`, else `ClassCastException` at runtime; they order + de-dupe by comparison, ignoring `equals()` / `hashCode()`.
7. Keep `(a.compareTo(b) == 0) == a.equals(b)` true, or `TreeSet` discards distinct items as duplicates and breaks the `Set` contract.This is a contract recommendation, not enforced — violations fail silently.


---
### **Fail-Fast vs. Fail-Safe Mechanism**

| Aspect | Fail-Fast | Fail-Safe / Weakly Consistent |
| :--- | :--- | :--- |
| **Core Concept & JDK Term** | Throws `ConcurrentModificationException` on mid-iteration edits. ("Best-effort" bug detector). | Iterates over a snapshot or weak view without exceptions. (Weakly consistent / COW). |
| **Mechanism & Exception Triggers** | Tracks structural changes via `modCount` vs `expectedModCount` checked on each `next()`. Only structural changes (`add`/`remove`) trigger it. set() doesn't | No comodification check; operates independently on a snapshot or concurrent view. No exception triggers. |
| **Collections & In-Loop Removal** | ArrayList, LinkedList, HashMap, HashSet, TreeMap, TreeSet, LinkedHashMap, LinkedHashSet, ArrayDeque, and PriorityQueue. <br>•Safe: Iterator.remove(), removeIf(), ListIterator.add()/set() | CopyOnWriteArrayList/Set, ConcurrentHashMap, ConcurrentLinkedQueue. <br>•COW iterator throws UnsupportedOperationException on remove(); CHM iterator supports it. |
| **Data Freshness & Thread Safety** | Live data until it throws ("consistent or fails"). <br>• **Not thread-safe** (CME can fire even single-threaded). | COW: true immutable snapshot; never sees writes made after iterator creation. CHM: weakly consistent — may or may not see concurrent writes, but returns each surviving element exactly once. <br>• Thread-safe. |
| **Performance, Memory & Risks** | Minimal overhead, no copying. <br>• *Risk:* Surprise CME crashes, infinite loops if shared raw. | COW: full $O(n)$ copy per write ($\rightarrow$ GC pressure). <br>• *Risk:* Memory blowup, stale reads, `size()` is an estimate. |
| **Best Fit / Use Case** | Single-threaded collections or external sync; used as an early bug signal. | Read-mostly shared lists (COW) or hot concurrent caches (`ConcurrentHashMap`). |

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
