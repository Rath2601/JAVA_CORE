## **Array** :
1. fixed number of homogeneous elements stored in contiguous memory locations.
2. Directly retrieving an element by index O(1) (efficient random access to elements).
3. Array of a parent class can have its subclass elements but its not vice versa.
4. final --> prevents reassigning the array reference but doesn’t make the array itself immutable.
```  
final int[] arr = new int[2]; 
arr = new int[2]; // this is not possible 
```
5. elements inside an array are always mutable.
```
arr[0] = 15;
arr[0] = 14; // array elements are mutable.
```
6. Array are allocated in heap memory.
7. Arrays are serializable by default. (rarely used in modern Apps, this is still significant for legacy systems and internal JVM frameworks)
   
---
## **List** :

1. Allow duplicates.
2. Insertion order is maintained.
3. Can have null values.
4. Allows elements to be accessed/inserted/deleted by their position.
5. Immutable lists (List.of, List.copyOf) don't allow null — but duplicates are allowed (unlike Set.of/Map.of)
6. Immutable lists → value-based objects. Must never synchronize on them or rely on == identity because the JVM may reuse instances based on value. Iteration order is not randomized (unlike Set.of/Map.of) — order is the contract
7. Lists created via List.of or List.copyOf are serializable if all elements are serializable
8. Collections.unmodifiableList is a live read-only view whose backing list can still change; List.copyOf is a defensive copy. Both shallow — elements stay mutable
9. Mutable elements are fine as list members (no hashing) — unlike Set/Map keys
---
#### Difference between ArrayList and LinkedList
| Feature                     | Array                                    | ArrayList                                 | LinkedList                                   |
|-----------------------------|------------------------------------------|-------------------------------------------|----------------------------------------------|
| **Data Structure**          | Fixed-size array                         | Resizable array                           | Doubly linked list                           |
| **Size**                    | Fixed                                    | Dynamic                                   | Dynamic                                      |
| **Access Time (get)**       | O(1) - Fast                              | O(1) - Fast                               | O(n) - Slower                                |
| **Insertion (add)**         | N/A - Fixed size                         | O(1) - at end, O(n) - at index            | O(1) - Adding elements anywhere              |
| **Deletion (remove)**       | N/A - Fixed size                         | O(n) - Due to shifting elements           | O(1) - Easy to remove from the middle        |
| **Memory Usage**            | Least memory per element                 | Less memory per element                   | More memory per element (due to pointers)    |
| **Traversal**               | Fast with index access                   | Faster with index access (random access)  | Slower for large lists (sequential access)   |
| **Best Suited For**         | Fixed number of elements, fast access    | Frequent access to elements by index      | Frequent insertions/deletions at any position|
| **Resize Capability**       | Not resizable                            | Automatically resizes. default capacity(10)                   | Automatically resizes                        |
| **Implements**              | N/A                                      | List, RandomAccess, Cloneable, Serializable | List, Deque, Cloneable, Serializable       |
| **Use Case**                | Static data where size is known          | Good for read-heavy operations            | Good for write-heavy operations              |

---
| Feature                                | ArrayList                                              | Vector                                                |
|----------------------------------------|--------------------------------------------------------|-------------------------------------------------------|
| **Synchronization**                    | ArrayList is not synchronized.                         | Vector is synchronized.                               |
| **Growth Strategy**                    | ArrayList increments 50% of current array size if the number of elements exceeds its capacity. | Vector increments 100%, meaning it doubles the array size if the total number of elements exceeds its capacity. |
| **Legacy Status**                      | ArrayList is not a legacy class. It was introduced in JDK 1.2. | Vector is a legacy class.                             |
| **Performance**                        | ArrayList is fast because it is non-synchronized.       | Vector is slow because it is synchronized. In a multithreading environment, it holds other threads in a runnable or non-runnable state until the current thread releases the lock of the object. |
| **Traversal**                          | ArrayList uses the `Iterator` interface to traverse the elements. Iterator check concurrent state changes and fails if there is any | Vector can use the `Iterator` interface or `Enumeration` interface to traverse the elements. Enumeration completely ignores concurrent state changes |
---
### **STACK** :
1. class represents a last-in-first-out (LIFO) stack of objects.
2. it has the methods that supports LIFO. but it also have List and Vector methods as it extends them. (it behaves like list as well)
3. Method-level synchronized (push() and pop()). Compound operations (e.g., isEmpty() followed by pop()) are not atomic and require external synchronization on the stack instance.

#### uses:
1. Undo mechanism (Text Editors)
2. Backtracking Algorithms (Maze solvers)
3. Depth-First Search (DFS)
4. Navigation in Web Browsers (history management).
---

## **Set**

1. No duplicates — uniqueness by equals()/hashCode(), except sorted sets where it's compare() == 0.
2. Set are internally map. except EnumSet (bit vector) and CopyOnWriteArraySet (COW list). HashSet→HashMap (dummy PRESENT value), LinkedHashSet→LinkedHashMap, TreeSet→TreeMap, ConcurrentSkipListSet→ConcurrentSkipListMap, newKeySet()→ConcurrentHashMap
3. One null element in HashSet/LinkedHashSet; TreeSet rejects under natural ordering (null-tolerant Comparator permits); EnumSet, Set.of(), Set.copyOf(), newKeySet(), ConcurrentSkipListSet, CopyOnWriteArraySet reject null
4. No index-based structure. Fast lookup/add/remove by value rather than position
5. Ordering: none (HashSet) / insertion (LinkedHashSet, no access-order option) / sorted (TreeSet, ConcurrentSkipListSet) / enum declaration (EnumSet) / randomized per JVM run (Set.of())
6. it is implemented with **mathematical set** logic.Supports set operations like union (addAll), intersection (retainAll), and difference (removeAll).
7. Never use a mutable element or mutable collection as a Set element. changing contents changes its hashCode, corrupting the bucket index — contains() returns false while size() still counts it
8. Iterator: fail-fast (HashSet, LinkedHashSet, TreeSet, EnumSet) / snapshot (CopyOnWriteArraySet) / weakly consistent (newKeySet(), ConcurrentSkipListSet). removeIf() is the correct conditional-delete idiom
9. Immutable sets → value-based objects. Randomized iteration order across executions. Must never synchronize on them or rely on == identity because the JVM may reuse instances based on value

CLASSES -> **HashSet**, **LinkedHashSet**, **TreeSet**.

| Feature | `HashSet` | `LinkedHashSet` | `TreeSet` |
| :--- | :--- | :--- | :--- |
| **Order of Elements** | No guaranteed order | Maintains insertion order | Sorted order (Natural or `Comparator`) |
| **Null Elements** | **1 `null` allowed** | **1 `null` allowed** | **No `null` allowed** (`NullPointerException`) |
| **Underlying Data Structure** | `HashMap` (Hash Table) | `LinkedHashMap` (Hash Table + Doubly-Linked List) | `TreeMap` (Red-Black Tree) |
| **Time Complexity** | $O(1)$ for `add`, `remove`, `contains`  $O(\log n)$ (Java 8+ treeification under high collisions) | $O(1)$ for `add`, `remove`, `contains` $O(\log n)$ (Java 8+ treeification under high collisions) | $O(\log n)$ for `add`, `remove`, `contains` |
| **Element Comparison Mechanism** | `hashCode()` and `equals()` | `hashCode()` and `equals()` | `compareTo()` (`Comparable`) or `compare()` (`Comparator`) |
| **Specialized Methods** | Standard Set operations | Access-order capabilities via underlying `LinkedHashMap` | Navigable operations (`first()`, `last()`, `higher()`, `lower()`, `subSet()`) |
| **Memory Overhead** | Low (Stores entries + hash buckets) | Medium (Extra pointers for doubly-linked list) | High (Tree node pointers: parent, left, right, color) |
| **Duplicates** | Does not allow | Does not allow | Does not allow |
| **Primary Use Case** | Fast lookups when order doesn't matter | Fast lookups preserving insertion/access sequence | Sorted datasets or range-based queries |


## **QUEUE**
1. FIFO by default; Deque is double-ended (both FIFO queue and LIFO stack). PriorityQueue orders by priority, not insertion
2. Most queues reject null (ArrayDeque, PriorityQueue, all concurrent/blocking queues) — null is the sentinel meaning "empty". Only LinkedList-as-Queue allows it
3. Two method families — throws vs. returns special value:
Insert: add(e) throws / offer(e) returns false
Remove: remove() throws / poll() returns null
Examine: element() throws / peek() returns null
4. BlockingQueue adds two more: put(e)/take() block, offer(e,t,u)/poll(t,u) time out
5. Deque as stack: push/pop/peek = addFirst/removeFirst/peekFirst. Use ArrayDeque, not Stack
6. ArrayDeque used as a stack iterates top-to-bottom; legacy Stack iterates bottom-to-top — a real migration bug
7. No index-based access — no get(i). Search/remove(Object) is 𝑂(𝑁)
8. PriorityQueue iteration is not sorted — only repeated poll() yields order; toString() prints raw heap-array order
9. Bounded vs unbounded is the key production property. Unbounded = no backpressure = OOM under a slow consumer. Unbounded: ConcurrentLinkedQueue, PriorityBlockingQueue, DelayQueue, LinkedTransferQueue, LinkedBlockingQueue by default. Bounded: ArrayBlockingQueue (always), LinkedBlockingQueue/LinkedBlockingDeque (only if capacity passed)
10. size() is 𝑂(𝑁) on ConcurrentLinkedQueue/Deque and LinkedTransferQueue — use isEmpty(). O(1) on ArrayBlockingQueue/LinkedBlockingQueue (AtomicInteger)
11. SynchronousQueue holds nothing — capacity 0, size() always 0, isEmpty() always true, iteration always empty
12. DelayQueue elements implement Delayed; poll() returns null while the head hasn't expired even though the queue is non-empty
13. Iterator: fail-fast (ArrayDeque, PriorityQueue) / weakly consistent (all concurrent and blocking queues). No queue offers snapshot iteration
14. Queue ordering is a contract of the implementation, not the interface — Queue itself guarantees nothing about order
15. LinkedList implements both List and Deque, which invites misuse — use ArrayList or ArrayDeque instead

| Attribute | PriorityQueue | ArrayDeque | LinkedList |
| :--- | :--- | :--- | :--- |
| **Primary Data Structure** | Resizable Array-based **Binary Min-Heap** | Resizable **Circular Array** | **Doubly Linked List** |
| **Ordering Behavior** | **Priority order** (Natural / `Comparator`) | **Insertion order** (FIFO / LIFO) | **Insertion order** (FIFO / LIFO / Index) |
| **Null Elements?** | **Forbidden** (Throws `NPE`) | **Forbidden** (Throws `NPE`) | **Allowed** |
| **Dynamic Resizing?** | **Yes** (Array re-allocation) | **Yes** (Array re-allocation) | **Yes** (Node allocation) |
| **Enqueue / Dequeue Complexity** | $O(\log n)$ | **Amortized $O(1)$** | **$O(1)$** |
| **Search / Contains Complexity** | $O(n)$ | $O(n)$ | $O(n)$ |
| **Random Access ($O(1)$ Index)** | **No** | **No** | **No** ($O(n)$ via sequential traversal) |
| **Memory Overhead** | **Low** (Array storage) | **Lowest** (Contiguous array, no node wrappers) | **Highest** (24–32 bytes node wrapper per element) |
| **Cache Locality** | **Moderate** | **High** (Contiguous memory blocks) | **Abysmal** (Pointer chasing across heap) |
| **Thread Safety** | **No** (Use `PriorityBlockingQueue`) | **No** (Use `ArrayBlockingQueue` / `CLQ`) | **No** (Use `LinkedBlockingQueue`) |
| **Primary Production Use Case** | Priority scheduling, Top- $K$ algorithms | Default **Queue** & **Stack** replacement | *Avoid in high-perf code* (Legacy / rare iteration node removal) |


## **Map**
### **Overview**
1. key-value pair. unique key with one null key (key like set)
2. Replaces `Dictionary`
3. keySet(), values(), entrySet() are live views, not copies — removal writes through, add unsupported, entry.setValue() mutates the map
4. map can't contain Itself as a Key/value: Syntax-wise it is valid Java. Architectural-wise, it violates the contract of Java Collections and leads to runtime crashes (StackOverflowError ex: from recursive hashCode()/equals())
5. Never use ANY mutable key or mutable collection (List, Set, Map) as a Map Key: Changing the collection's contents changes its hashCode, corrupting the bucket index and making the key unretrievable.
6. Hashtable, TreeMap, ConcurrentHashMap, ConcurrentSkipListMap, EnumMap, Map.of(), Map.copyOf() will not allow null key
7. Lookup Optimization:hash(K) $\rightarrow$ Bucket Index $\rightarrow$ hash == node.hash $\rightarrow$ key == node.key $\rightarrow$ key.equals(node.key)
8. null Key Handling:key == null $\rightarrow$ hash = 0 $\rightarrow$ directly routes to Bucket 0 (bypasses hashCode() and .equals()).
9. Immutable map don't allow null keys/values; duplicate keys throw IllegalArgumentException at construction (unlike HashMap, which silently overwrites)
10. Immutable maps -> value based objects. have randomized iteration orders across different executions. must never synchronize on them or rely on == identity because the JVM may reuse instances based on value
11. Maps created via Map.of, Map.ofEntries, or Map.copyOf are serializable if all keys and values are serializable
    
---
| **Aspect**                | **HashMap**                                 | **TreeMap**                                | **LinkedHashMap**                           |
|---------------------------|---------------------------------------------|--------------------------------------------|--------------------------------------------|
| **Ordering**               | No ordering (unordered)                    | Ordered according to **natural order** of keys or a custom comparator | Maintains insertion order of elements |
| **Null Keys/Values**       | Allows one `null` key and multiple `null` values | Does not allow `null` keys (throws `NullPointerException`), but allows `null` values | Allows one `null` key and multiple `null` values |
| **Performance (Time Complexity)** | O(1) for basic operations (put, get)    | O(log n) for most operations (put, get, remove) | O(1) for basic operations (put, get)       |
| **Order of Iteration**     | Unspecified                                 | Sorted order (ascending order of keys)     | Iteration order is the same as insertion order |
| **Comparator**             | Does not require a comparator for ordering | Requires a comparator for custom ordering or uses natural order | No comparator, but maintains insertion order |
| **Thread Safety**          | Not thread-safe (use `Collections.synchronizedMap()` for synchronization) | Not thread-safe (use `Collections.synchronizedMap()` for synchronization) | Not thread-safe (use `Collections.synchronizedMap()` for synchronization) |
| **Usage**                  | Ideal for general-purpose maps that don't require ordering | Ideal for maps where sorting or range queries are needed | Ideal when insertion order needs to be preserved |
| **Space Overhead**         | Less overhead                               | Higher overhead due to tree structure      | Higher overhead due to maintaining insertion order |
| **Iterator**               | Allows `null` key, but iterators are not ordered | Sorted iterators based on key order        | Iterators maintain the insertion order of elements |
| **KeySet**                  | Unordered key set                          | Sorted key set                             | Key set is in insertion order              |
