# C++20 Thread Management: `std::jthread`, Joiners, and Stop Tokens

## 🧵 1. `std::thread` Basics

`std::thread` allows you to create threads easily, but you must either:
- **join** the thread (wait for it to finish), or  
- **detach** it (run independently).

If you forget to do either, your program will **terminate**.

```cpp
#include <thread>
#include <iostream>

void worker() {
    std::cout << "Hello from worker thread\n";
}

int main() {
    std::thread t(worker);
    // Forgot t.join() or t.detach()
    // → std::terminate() called
}
```

---

## 🧹 2. RAII Joiner Pattern

Wrap a `std::thread` inside an RAII class to automatically join on destruction.

```cpp
#include <thread>
#include <iostream>

class thread_joiner {
    std::thread& t_;
public:
    explicit thread_joiner(std::thread& t) : t_(t) {}
    ~thread_joiner() {
        if (t_.joinable()) {
            t_.join();
        }
    }
};

void worker() {
    std::cout << "Work done!\n";
}

int main() {
    std::thread t(worker);
    thread_joiner joiner(t); // ensures join at scope exit
}
```

✅ **Advantages:**
- Prevents forgetting to join.
- Follows RAII (Resource Acquisition Is Initialization).

⚠️ **Disadvantages:**
- Requires manual helper class.
- No built-in stop mechanism.

---

## 🚀 3. `std::jthread` (C++20)

`std::jthread` is the modern replacement for `std::thread`. It:
- Automatically **joins** on destruction.
- Adds **cooperative cancellation** with `std::stop_token`.

```cpp
#include <thread>
#include <iostream>

void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::cout << "Working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
    std::cout << "Worker stopping.\n";
}

int main() {
    std::jthread t(worker);  // automatically joins
    std::this_thread::sleep_for(std::chrono::seconds(1));
    t.request_stop();        // asks thread to stop
}
```

| Feature | `std::thread` | Joiner Pattern | `std::jthread` |
|----------|----------------|----------------|----------------|
| Auto-join | ❌ | ✅ | ✅ |
| Stop mechanism | ❌ | ❌ | ✅ |
| Introduced | C++11 | Custom | C++20 |

---

## 🧠 4. `std::stop_token` and `std::stop_source`

### The Problem (Pre-C++20)
Before, stopping threads required custom atomic flags:
```cpp
std::atomic<bool> stop_flag{false};
```

### The Solution (C++20)
C++20 introduces a coordinated stop system:

| Type | Purpose |
|------|----------|
| `std::stop_source` | Used to **request** a stop |
| `std::stop_token` | Used to **check** for stop requests |
| `std::stop_callback` | Runs a function when a stop is requested |

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <stop_token>

void worker(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::cout << "Working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
    }
    std::cout << "Worker received stop request.\n";
}

int main() {
    std::stop_source source;
    std::stop_token token = source.get_token();
    std::jthread t(worker, token);

    std::this_thread::sleep_for(std::chrono::seconds(2));
    source.request_stop(); // Signal stop
}
```

---

## 🧩 5. `std::stop_callback` Example

```cpp
#include <iostream>
#include <stop_token>

void cleanup() {
    std::cout << "Cleanup called on stop request!\n";
}

int main() {
    std::stop_source source;
    std::stop_token token = source.get_token();
    std::stop_callback cb(token, cleanup);

    source.request_stop();
}
```

Output:
```
Cleanup called on stop request!
```

---

## 🔄 6. Sharing Tokens Across Threads

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <stop_token>
#include <chrono>

void worker(int id, std::stop_token st) {
    while (!st.stop_requested()) {
        std::cout << "Worker " << id << " running\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
    std::cout << "Worker " << id << " stopping\n";
}

int main() {
    std::stop_source src;
    std::stop_token token = src.get_token();

    std::vector<std::jthread> threads;
    for (int i = 0; i < 3; ++i)
        threads.emplace_back(worker, i, token);

    std::this_thread::sleep_for(std::chrono::seconds(1));
    src.request_stop();
}
```

---

## 🧾 7. Summary Table

| Concept | Description | Typical Owner |
|----------|--------------|----------------|
| `std::stop_source` | The controller that requests stop | Thread creator |
| `std::stop_token` | The observer that checks stop status | Worker thread |
| `std::stop_callback` | Function triggered on stop | Either side |

---

## ⚖️ 8. Comparison of Stop Mechanisms

| Approach | Type Safe | Thread-safe | Callbacks | Standardized |
|-----------|------------|-------------|------------|---------------|
| Atomic flag | ✅ | ✅ | ❌ | ❌ |
| Condition variable | ✅ | ✅ | ✅ (manual) | ❌ |
| `std::stop_token` System | ✅ | ✅ | ✅ | ✅ |

---

## 🧩 9. Integration with `std::jthread`

`std::jthread` automatically owns a `std::stop_source`, so you can request a stop directly:

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void worker(std::stop_token st) {
    while (!st.stop_requested()) {
        std::cout << "Working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
    }
    std::cout << "Stopped.\n";
}

int main() {
    std::jthread t(worker);
    std::this_thread::sleep_for(std::chrono::seconds(1));
    t.request_stop(); // Stop via internal stop_source
}
```

---

## ✅ Conclusion

- `std::jthread` simplifies thread management with automatic join and cancellation.
- `std::stop_source` and `std::stop_token` provide a standardized, thread-safe way to request stops.
- Prefer `std::jthread` over `std::thread` when using C++20 or newer.
