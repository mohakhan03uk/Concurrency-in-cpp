# 🧩 C++ Atomic `compare_exchange` — Deep Dive

## Overview

`compare_exchange` is one of the most important atomic operations in C++’s `<atomic>` library.  
It performs a **compare-and-swap (CAS)** — an atomic read-modify-write used for lock-free programming.

There are two variants:
- `compare_exchange_weak`
- `compare_exchange_strong`

---

## Function Signatures

```cpp
bool compare_exchange_weak(T& expected, T desired,
    std::memory_order success,
    std::memory_order failure);

bool compare_exchange_strong(T& expected, T desired,
    std::memory_order success,
    std::memory_order failure);
```

### Parameters

| Parameter | Description |
|------------|--------------|
| `expected` | The value expected to be currently stored. On failure, updated to the actual value. |
| `desired` | The new value to store if the atomic equals `expected`. |
| `success` | Memory order used if the exchange succeeds. |
| `failure` | Memory order used if the exchange fails (cannot be stronger than acquire). |

**Return:**  
- `true` → swap succeeded (atomic value matched `expected`).  
- `false` → swap failed (atomic value was different).

---

## `compare_exchange_strong()`
- `compare_exchange_strong()` performs a non-spurious atomic compare-and-swap.
- It will only fail if the actual value of the atomic object does not equal the expected value.

#### Behavior
- Compares the atomic variable’s value with expected.
- If equal → replaces it with desired atomically, returns true.
- If not equal → writes the current value into expected, returns false.
- Never fails spuriously — the failure reason is always a real mismatch.

#### Example: Basic Usage 

```cpp
#include <iostream>
#include <atomic>

int main() {
    std::atomic<int> value{10};
    int expected = 10;

    if (value.compare_exchange_strong(expected, 20)) {
        std::cout << "Exchange succeeded, new value = " << value << "\n";
    } else {
        std::cout << "Exchange failed, expected was " << expected << "\n";
    }
}
```

##### Output
```
Exchange succeeded, new value = 20
```

If the atomic’s value was not 10, the operation fails, and `expected` is updated to the actual value.

#### Performance
- Slightly slower on some architectures because it enforces strong guarantees (no spurious fail).
- Recommended when you are not looping, e.g. in single-shot updates or synchronization points.
#### Common use cases
- Reference counting increments/decrements.
- Once-only updates, e.g. state flags.
- When correctness must not depend on retry loops.
---

## `compare_exchange_weak`
- `compare_exchange_weak` performs the same logical operation, but it is allowed to fail spuriously — meaning it can return false even when the atomic value equals expected.

### Why Weak Exists
- Some CPU architectures (especially ARM, PowerPC, Itanium) implement `compare-and-swap` using load–linked/store–conditional (LL/SC) pairs.
- These can fail for reasons other than value mismatch — e.g. cache invalidation or contention.
- So `compare_exchange_weak()` gives the compiler permission to map directly to that instruction, avoiding expensive retries hidden inside the library implementation.

#### Behavior Summary

| Event                    | Description                                       |
| ------------------------ | ------------------------------------------------- |
| Value matches `expected` | May **still fail spuriously**, returning `false`. |
| Value does not match     | Always fails (and updates `expected`).            |
| On success               | Updates value atomically to `desired`.            |

#### Example: Basic Usage
```cpp
  std::atomic<int> x{0};
  int expected = 0;
  
  do {
      expected = 0; // Reset expected for retry
  } while (!x.compare_exchange_weak(expected, expected + 1));

```
#### Performance
- Faster because it allows hardware-friendly fail semantics.
- Usually used inside a loop to retry if failure occurs.

## Weak vs Strong

| Feature           | `compare_exchange_weak`          | `compare_exchange_strong`         |
| ----------------- | -------------------------------- | --------------------------------- |
| Spurious Failures | ✅ May happen                     | ❌ Never happens                   |
| Performance       | Usually faster                   | Slightly slower                   |
| Hardware Mapping  | Matches LL/SC or CAS directly    | May insert retry loops internally |
| Use Pattern       | Use in retry loops               | Use for single-shot operations    |
| Failure Cause     | Value mismatch or hardware retry | Only value mismatch               |
| Common Usage      | Lock-free loops, spinlocks       | Simple atomic state updates       |



## Choosing Between Them
| Scenario                               | Recommended               |
| -------------------------------------- | ------------------------- |
| Loop-based algorithm                   | `compare_exchange_weak`   |
| Single update (once-only)              | `compare_exchange_strong` |
| High contention expected               | `weak` (less overhead)    |
| Readability or portability prioritized | `strong`                  |


---

## Real-World Uses

- **Lock-free stacks, queues, and lists**
- **Reference counting**
- **Spinlocks**
- **ABA-safe synchronization algorithms**

---

## Memory Ordering

You can control how memory synchronization behaves:

```cpp
value.compare_exchange_strong(expected, desired,
    std::memory_order_acq_rel,
    std::memory_order_relaxed);
```

### Common Orders

| Memory Order | Meaning |
|---------------|----------|
| `memory_order_relaxed` | No synchronization, only atomicity. |
| `memory_order_acquire` | Reads after this can’t move before it. |
| `memory_order_release` | Writes before this can’t move after it. |
| `memory_order_acq_rel` | Combines both acquire and release. |
| `memory_order_seq_cst` | Sequential consistency (default, strictest). |

---

## Summary

| Concept | Description |
|----------|--------------|
| Type | Atomic read–compare–write |
| Returns | `true` if swapped, `false` otherwise |
| Updates `expected` | Yes (on failure) |
| Weak vs Strong | Weak may fail spuriously |
| Common Use | Lock-free synchronization |
| Memory Order | Controls visibility and reordering |

---

## References

- [cppreference.com — `std::atomic::compare_exchange`](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)
- [C++ Standard §33.5.8 — Atomics and Operations](https://eel.is/c++draft/atomics)
