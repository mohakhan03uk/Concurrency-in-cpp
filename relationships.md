# C++ Atomic Thread Relationships

## 1. Happens-Before Relationship
### Definition: 
  If operation A happens-before operation B, then all side effects (writes) of A are visible to B.
### Purpose: 
  Ensures memory consistency between threads.
### Atomic role:
  Atomic operations can establish happens-before relationships.
### Example
```cpp
    #include <atomic>
    #include <thread>
    #include <iostream>
    
    std::atomic<int> data{0};
    std::atomic<bool> ready{false};
    
    void producer() {
        data.store(42, std::memory_order_relaxed); // write
        ready.store(true, std::memory_order_release); // release
    }
    
    void consumer() {
        while (!ready.load(std::memory_order_acquire)) {} // acquire
        std::cout << data.load(std::memory_order_relaxed) << "\n"; // sees 42
    }
    
    int main() {
        std::thread t1(producer);
        std::thread t2(consumer);
        t1.join();
        t2.join();
    }
```
- `release` (producer) + `acquire` (consumer) → establishes **happens-before**, so `data = 42` is visible to the consumer.

## 2. Synchronized-With Relationship

### Definition
A **synchronized-with** relationship exists between two atomic or mutex-based operations in *different threads* when one performs a **release** operation and another performs a **corresponding acquire** operation on the same synchronization object.

Formally:
> If A is a release operation and B is an acquire operation on the same atomic variable, then **A synchronizes-with B**.

This ensures all writes before A become visible to reads after B.

### Example
```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> data{0};
std::atomic<bool> ready{false};

void producer() {
    data.store(42, std::memory_order_relaxed);      // write data
    ready.store(true, std::memory_order_release);   // release store
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {} // acquire load
    std::cout << data.load(std::memory_order_relaxed) << "\n"; // guaranteed to see 42
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
}
```

**Explanation:**
- `ready.store(release)` synchronizes-with `ready.load(acquire)`.
- All modifications to memory before the release (e.g., `data.store(42)`) become visible after the acquire.

### Visualization
```
Thread 1: [write data] --sb--> [release ready]
                               |
                               | sw (synchronized-with)
                               v
Thread 2: [acquire ready] --sb--> [read data]
```
- `sb` = sequenced-before
- `sw` = synchronized-with

---

## 2. Inter-Thread Happens-Before (ITHB)

### Definition
**Inter-thread-happens-before (ITHB)** is the transitive closure of **sequenced-before** (within a thread) and **synchronized-with** (across threads) relationships.

**Sequenced-before:** order within a single thread (program order). The sequenced-before relationship defines ***the order of execution of operations*** **within a single thread**, as determined by the program. 
  - It is a strict, intra-thread ordering.
  - If operation A is sequenced-before B, then:
      - All effects of A are visible to B.
      - B cannot be observed to happen before A by that thread.
      - There is no overlap; A is fully completed before B starts (from that thread’s view).
    
**Synchronized-with:** order across threads via atomic/mutex operations.

Formally:
> If A is sequenced-before X in thread 1, and X is synchronized-with Y in thread 2, and Y is sequenced-before B in thread 2, then **A inter-thread-happens-before B**.

This means all side effects of A are visible to B.

#### Transitive Rule:
```cpp 
sequenced-before (in thread 1) → release → synchronized-with → acquire (in thread 2) → sequenced-before (thread 2)
```
###### This gives inter-thread happens-before, which guarantees:
- Atomicity of operations is respected.
- Memory visibility: writes before the release are visible after the acquire.
###### Example diagram:
```cpp
Thread 1: [write data] --sb--> [release ready]
                               |
                               | sw
                               v
Thread 2: [acquire ready] --sb--> [read data]
```
- `sb =` sequenced-before
- `sw =` synchronized-with
- The read of data is guaranteed to see the write (42) because sb + sw forms ITHB.

### Example
```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> data{0};
std::atomic<bool> flag{false};

void writer() {
    data.store(100, std::memory_order_relaxed);   // (1) write data
    flag.store(true, std::memory_order_release);  // (2) release
}

void reader() {
    while (!flag.load(std::memory_order_acquire)) {} // (3) acquire
    std::cout << data.load(std::memory_order_relaxed) << "\n"; // (4) read data
}

int main() {
    std::thread t1(writer);
    std::thread t2(reader);
    t1.join();
    t2.join();
}
```

### ITHB Chain
```
(1) data.store --sb--> (2) flag.store(release)
                           |
                           | sw (synchronized-with)
                           v
                   (3) flag.load(acquire) --sb--> (4) data.load
```
Thus, (1) **inter-thread-happens-before** (4).

### Guarantees
- All writes before the release in one thread are visible after the acquire in another.
- Prevents data races on synchronized data.

---
#### Key Points

| Concept                         | Scope                                                    | Guarantees                                | Example                              |
| ------------------------------- | -------------------------------------------------------- | ----------------------------------------- | ------------------------------------ |
| **Synchronized-with**           | Between two operations (typically atomic) across threads | Visibility of previous writes             | release store → acquire load         |
| **Inter-thread-happens-before** | Across threads                                           | Transitive memory ordering across threads | sequenced-before + synchronized-with |



## 3. Summary Table

| Relationship | Between | Established by | Guarantees |
|---------------|----------|----------------|-------------|
| **Sequenced-before** | Operations in one thread | Program order | Defines intra-thread execution order |
| **Synchronized-with** | Operations in different threads | Release/acquire, lock/unlock | Visibility across threads |
| **Inter-thread-happens-before** | Across threads (transitive) | Combination of `sb` and `sw` | Consistent memory view between threads |

---

## 4. Important Notes
- `memory_order_relaxed` **does not** establish synchronized-with or ITHB.
- `memory_order_release` and `memory_order_acquire` are minimal primitives for creating ITHB.
- `memory_order_seq_cst` creates a **global total order** (strongest guarantee).

---

## 5. Visual Summary
```
Thread 1:   A (write) --sb--> B (release)
                               |
                               | sw (synchronized-with)
                               v
Thread 2:   C (acquire) --sb--> D (read)

=> A inter-thread-happens-before D
```

This ensures any write before the release in one thread is visible to any read after the acquire in another.

