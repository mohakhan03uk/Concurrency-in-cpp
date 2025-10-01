# std::unique_lock and std::try_to_lock - Discussion and Examples

## Introduction
This document summarizes about `std::unique_lock`, its usage.

---

## What is `std::unique_lock`?
`std::unique_lock` is a C++ Standard Library class template that provides a more flexible way to manage mutex locking than `std::lock_guard`.

### Differences from `std::lock_guard`
- **`std::lock_guard`**:
  - Always locks immediately upon construction.
  - Always unlocks on destruction.
  - Very lightweight.

- **`std::unique_lock`**:
  - Can defer locking, try locking, or adopt an already-locked mutex.
  - Allows unlocking and relocking multiple times.
  - Supports move semantics.
  - Required by `std::condition_variable`.

### Constructors
```cpp
std::unique_lock<std::mutex> lock1(mtx);                  // lock immediately
std::unique_lock<std::mutex> lock2(mtx, std::defer_lock); // construct but don’t lock
std::unique_lock<std::mutex> lock3(mtx, std::try_to_lock);// try_lock(), may fail
std::unique_lock<std::mutex> lock4(mtx, std::adopt_lock); // mutex already locked
```
---
## Example 0: Defer and Lock Later
```cpp
  std::mutex mtx;
  
  void work() {
      std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
      // do something before locking
      lock.lock();
      // critical section
      lock.unlock();
      // do more work without lock
  }

```
---

## Example 1: One-Shot `std::try_to_lock`
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex mtx;

void worker(int id) {
    // Try to acquire lock without waiting
    std::unique_lock<std::mutex> lock(mtx, std::try_to_lock);

    // owns_lock() must be checked after using std::try_to_lock
    if (lock.owns_lock()) {
        std::cout << "Thread " << id << " got the lock!\n";
        std::this_thread::sleep_for(std::chrono::seconds(2)); // simulate work
        std::cout << "Thread " << id << " releasing lock.\n";
    } else {
        std::cout << "Thread " << id << " could NOT get the lock.\n";
    }
}

int main() {
    std::thread t1(worker, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // small delay
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
    return 0;
}
```

### Sample Output
```
Thread 1 got the lock!
Thread 2 could NOT get the lock.
Thread 1 releasing lock.
```

---

## Example 2: Retrying with Timeout (Manual Loop)
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::mutex mtx;

void worker(int id) {
    using namespace std::chrono_literals;

    std::unique_lock<std::mutex> lock(mtx, std::defer_lock); // don’t lock yet

    auto deadline = std::chrono::steady_clock::now() + 3s; // try for 3 seconds

     // owns_lock() must be checked after using std::try_to_lock
    while (!lock.owns_lock() && std::chrono::steady_clock::now() < deadline) {
        lock.try_lock(); // non-blocking attempt
        if (!lock.owns_lock()) {
            std::cout << "Thread " << id << " failed, retrying...\n";
            std::this_thread::sleep_for(200ms); // backoff before retry
        }
    }
    // owns_lock() must be checked after using std::try_to_lock
    if (lock.owns_lock()) {
        std::cout << "Thread " << id << " acquired the lock!\n";
        std::this_thread::sleep_for(2s); // simulate work
        std::cout << "Thread " << id << " releasing lock.\n";
    } else {
        std::cout << "Thread " << id << " gave up after timeout.\n";
    }
}

int main() {
    std::thread t1(worker, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // ensure t1 starts first
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
    return 0;
}
```

### Sample Output
```
Thread 1 acquired the lock!
Thread 2 failed, retrying...
Thread 2 failed, retrying...
Thread 2 acquired the lock!
Thread 1 releasing lock.
Thread 2 releasing lock.
```

---

## Notes
- `.owns_lock()` must be checked after using `std::try_to_lock`.
- For built-in timeout support, consider using **`std::timed_mutex`** with `try_lock_for()` or `try_lock_until()` instead of writing a retry loop.
- While try_lock_for() waits for a relative duration, try_lock_until() waits until a specific time point.
---
## Example 3: Using std::timed_mutex with try_lock_for
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::timed_mutex tmtx;

void worker(int id) {
    using namespace std::chrono_literals;

    std::unique_lock<std::timed_mutex> lock(tmtx, std::defer_lock); // start unlocked

    if (lock.try_lock_for(3s)) { // wait up to 3 seconds
        std::cout << "Thread " << id << " acquired the lock!\n";
        std::this_thread::sleep_for(2s); // simulate work
        std::cout << "Thread " << id << " releasing lock.\n";
    } else {
        std::cout << "Thread " << id << " gave up after 3 seconds.\n";
    }
}

int main() {
    std::thread t1(worker, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // ensure t1 runs first
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
    return 0;
}
```
### How it works
- t1 locks the timed_mutex and holds it for 2 seconds.
- t2 tries to acquire the lock with try_lock_for(3s).
- If t1 releases within 3 seconds → t2 succeeds.
- Otherwise → t2 times out and fails.
  
### Sample output 
```
Thread 1 acquired the lock!
Thread 2 acquired the lock!
Thread 1 releasing lock.
Thread 2 releasing lock.
```
###### (or, if t1 holds too long:)
```
Thread 1 acquired the lock!
Thread 2 gave up after 3 seconds.
Thread 1 releasing lock.
```

## Example 4: Using try_lock_until()
```
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::timed_mutex tmtx;

void worker(int id) {
    using namespace std::chrono;

    std::unique_lock<std::timed_mutex> lock(tmtx, std::defer_lock); // start unlocked

    // Calculate an absolute deadline: 3 seconds from now
    auto deadline = steady_clock::now() + seconds(3);

    if (lock.try_lock_until(deadline)) {
        std::cout << "Thread " << id << " acquired the lock!\n";
        std::this_thread::sleep_for(seconds(2)); // simulate work
        std::cout << "Thread " << id << " releasing lock.\n";
    } else {
        std::cout << "Thread " << id << " could not acquire lock before deadline.\n";
    }
}

int main() {
    std::thread t1(worker, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // ensure t1 runs first
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
    return 0;
}
```
### How it works?
- t1 acquires the timed_mutex immediately.
- t2 tries to acquire the lock until a deadline = now + 3s.
- If t1 releases within that time → t2 succeeds.
- If not → t2 fails at the deadline.

### Sample output 
```
Thread 1 acquired the lock!
Thread 2 acquired the lock!
Thread 1 releasing lock.
Thread 2 releasing lock.
```
###### (or, if t1 holds too long:)
```
Thread 1 acquired the lock!
Thread 2 could not acquire lock before deadline.
Thread 1 releasing lock.
```
