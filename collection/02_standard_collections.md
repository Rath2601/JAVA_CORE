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

1. Does not allow duplicates.
2. Insertion order is not maintained.(generally)
   * HashSet (doesn't guarantee insertion order or any order)
   * TreeSet (sort based on natural sorting)
   * LinkedHashSet (preserve insertion order)
4. Can have one null value. (generally)
5. Sets do not maintain an index-based structure.primary purpose of a Set is to maintain unique elements without duplicates, not to store elements in a particular order.
6. Sets are designed for fast lookups (like contains()), additions, and removals **based on the value** itself **rather than its position**.
7. it is implemented with **mathematical set** logic.Supports set operations like union (addAll), intersection (retainAll), and difference (removeAll).

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
- A `Map` is a collection of objects that map **keys** to **values**.
- **Key Characteristics**:
  - **Key-Value Mapping**: Each key maps to exactly one value.
  - **No Duplicate Keys**: Keys must be unique.
  - **Replaces `Dictionary`**: `Map` takes the place of the older `Dictionary` abstract class.
  - Allows a map's contents to be viewed as:
    - A **Set** of keys.
    - A **Collection** of values.
    - A **Set** of key-value mappings.
---
### **Order Behavior in Common Implementations**
1. **HashMap**:
   - Does not maintain order.
   - Uses hashing for storage and retrieval.
2. **TreeMap**:
   - Maintains keys in **natural order**.
   - Does not allow `null` keys (throws `NullPointerException`).
3. **LinkedHashMap**:
   - Maintains keys in **insertion order**.
---
### **Hashing Considerations** -[This is common for all Hashing based collection classes]
- All hashing-based map implementations should override:
  - `hashCode` method.
  - `equals` method.
    
- **Mutable Objects as Keys**:
  - Should be avoided as they can cause unexpected behavior.
  - Example:
    ```java
    Map<StringBuilder, String> map = new HashMap<>();
    StringBuilder key = new StringBuilder("example");
    map.put(key, "value");
    key.append("changed"); // Modifies the key's state
    System.out.println(map.get(key)); // Might return null
    ```
---
### **Self-Referential Maps**
- **A Map Cannot Contain Itself as a Key**:
  - Causes `StackOverflowError`.
- **A Map Can Contain Itself as a Value**:
  - Allowed but must be used cautiously to avoid issues in:
    - `clone()`
    - `equals()`
    - `hashCode()`
    - `toString()`
---
### **Unsupported Operations**
- If a map does not allow modification:
  - Destructive methods (`put`, `remove`, `clear`) throw `UnsupportedOperationException`.
  - Optional behavior: Even if the operation has no effect (e.g., `putAll` on an empty map), the exception **may** still be thrown.
---
### **Standard Constructors**
1. **No-args Constructor**:
   - Creates an empty map.
   - Example:
     ```java
     Map<String, String> emptyMap = new HashMap<>();
     ```
2. **Copy Constructor**:
   - Creates a new map by copying the key-value pairs from an existing map.
   - Example:
     ```java
     Map<String, String> original = new HashMap<>();
     original.put("A", "Apple");
     Map<String, String> copy = new HashMap<>(original);
     ```
---
### **Behavior of Keys and Values**
- **Restrictions**:
  - Some map implementations (e.g., `Hashtable`) do not allow:
    - `null` keys or values.
    - Keys/values of incompatible types (`ClassCastException`).
- **Behavior**:
  - May throw exceptions like:
    - `NullPointerException`
    - `ClassCastException`
  - Query methods (`containsKey`, `containsValue`) may:
    - Return `false` for ineligible keys/values.
    - Throw exceptions in some implementations.
---
### **Relation to `equals` and `hashCode`**
- Many methods in `Map` rely on `equals` and `hashCode`:
  - Example: `containsKey` checks if a key exists.
    - If `key == null`: Looks for a `null` key.
    - Otherwise: Uses `key.equals(k)` to match.
- **Optimizations**:
  - Hashing implementations (e.g., `HashMap`) compare hash codes first before invoking `equals`.
---
### **Unmodifiable Maps**
- Created using:
  - `Map.of`
  - `Map.ofEntries`
  - `Map.copyOf`
- **Characteristics**:
  - **Immutable**: Modifying operations (`put`, `remove`) throw `UnsupportedOperationException`.
    ```
    Map test = Map.of(1, "Rathna",2,"Sathya");
    test.put(2, "Keerthi");
    ```
  - **Disallow `null` keys and values** (`NullPointerException`).
    ```
    Map test = Map.of(1, "Rathna",2,"Sathya", null,null); // NullPointerException
    ```
  - **Reject duplicate keys at creation** (`IllegalArgumentException`).
    ```
    Map test = Map.of(1, "Rathna",2,"Sathya", 2,"Rathna"); //IllegalArgumentException
    ```
  - **Iteration order is unspecified.**
  - Value-based:
    - Treat logically equal instances as interchangeable.(logically equal map, but distinct)
    - Avoid using them for synchronization.
      [**If synchronization is required, a synchronized or concurrent map is a better choice because they are explicitly designed for multi-threaded environments.**]
---
### **Serialization**
- Maps created via `Map.of`, `Map.ofEntries`, or `Map.copyOf` are **serializable** if all keys and values are serializable.
---
### **Key Notes on Behavior**
1. **Poor `hashCode` or `equals` Implementation**:
   - Leads to inconsistent behavior (e.g., `containsKey` may fail).
2. **Recursive Traversal in Self-Referential Maps**:
   - Methods like `clone`, `equals`, `hashCode`, and `toString` may fail.
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
