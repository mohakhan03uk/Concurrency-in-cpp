
# ThreadGuard & Safe `std::thread` Joining in C++

- We can call the detach fuction as soon as we launched the thread, as detach() does not block the the calling thread.
- In some occasions we can not call join() as soon as we lauch a thread, as join call block the calling thread.
- So we need see how to call join() on thread to avoid std::terminate() call happening.

---

## What this document covers
- Why **not joining** a `std::thread` can crash your program  
- The compiler error you saw and why it happened  
- A corrected `ThreadGuard` RAII wrapper (and why RAII is preferred)  
- A complete, working example (`Threads.cpp`) that calls different behaviors based on a command-line argument  
- How to compile and run on common toolchains  
- An optional **owning** `scoped_thread` alternative and modern C++ note (`std::jthread`)

---

##  `ThreadGuard.h`

```cpp
// ThreadGuard.h
#ifndef THREAD_GUARD_H
#define THREAD_GUARD_H

#include <thread>

class ThreadGuard {
    std::thread& _t;          // reference to an existing thread

public:
    explicit ThreadGuard(std::thread& t) : _t(t) {}

    ~ThreadGuard() {
        if (_t.joinable())
            _t.join();
    }

    // prevent copying and assignment
    ThreadGuard(ThreadGuard const&) = delete;
    ThreadGuard& operator=(ThreadGuard const&) = delete;

    // optionally forbid moving as well (since we hold a reference)
    ThreadGuard(ThreadGuard&&) = delete;
    ThreadGuard& operator=(ThreadGuard&&) = delete;
};

#endif // THREAD_GUARD_H
```

**Notes**
- This class holds a **reference** to a `std::thread`. It does **not own** the thread object — the `std::thread` object must outlive the `ThreadGuard`.
- The destructor calls `join()` if the thread is still joinable: this ensures we don't trigger `std::terminate()` in the `std::thread` destructor.

---

## Full example: `Threads.cpp` (conditional behavior based on `argv[1]`)

```cpp
// Threads.cpp
#include <iostream>
#include <thread>
#include <stdexcept>
#include <chrono>
#include "ThreadGuard.h"

void createProblem() {
    std::cout << "Hello from createProblem!\n";
    throw std::runtime_error("this is a runtime error");
}

void hello() {
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    std::cout << "Hello from hello!\n";
}

void skippingJoinCall() {
    std::cout << "Hello from skippingJoinCall!\n";
    std::thread t(hello);
    try {
        createProblem(); // throws before join()
        t.join();        // never reached
    } catch(...) {
        // thread still joinable here -> unsafely ignored
    }
    std::cout << "Bye from skippingJoinCall!\n";
}

void callJoinInCatchBlock() {
    std::cout << "Hello from callJoinInCatchBlock!\n";
    std::thread t(hello);
    try {
        createProblem(); // throws before join()
        t.join();        // never reached
    } catch(...) {
        // safe: ensure the thread is joined
        if (t.joinable())
            t.join();
    }
    std::cout << "Bye from callJoinInCatchBlock!\n";
}

void callJoinInThreadGuard() {
    std::cout << "Hello from callJoinInThreadGuard!\n";
    std::thread t(hello);
    ThreadGuard tg(t); // RAII: ensures join() in destructor
    try {
        createProblem(); // throws
    } catch(...) {
        // nothing required here; ThreadGuard will join when it is destroyed
    }
    std::cout << "Bye from callJoinInThreadGuard!\n";
}

int main(int argc, char* argv[]) {
    std::cout << "Welcome to Concurrency in C++\n";

    if (argc < 2) {
        std::cerr << "Usage: " << argv[0]
                  << " [1=skipJoin | 2=catchJoin | 3=threadGuard]\n";
        return 1;
    }

    int choice = std::stoi(argv[1]);

    switch (choice) {
        case 1:
            skippingJoinCall();
            break;
        case 2:
            callJoinInCatchBlock();
            break;
        case 3:
            callJoinInThreadGuard();
            break;
        default:
            std::cerr << "Invalid choice: " << choice << "\n";
            return 1;
    }

    std::cout << "========== Program ended cleanly ==========\n";
    return 0;
}
```

---

## Expected behaviour and sample outputs

- **Choice 1 (`./Threads 1`) — `skippingJoinCall()`**
  - `createProblem()` throws before `t.join()` is called.
  - The catch block **does not join** the thread.
  - When the local `std::thread t` is destroyed while still joinable, the C++ runtime calls `std::terminate()` and the program exits immediately.
  - You will likely see:
    ```
    Welcome to Concurrency in C++
    Hello from skippingJoinCall!
    Hello from createProblem!
    <program terminates via std::terminate — no 'Bye...' or final log printed>
    ```

- **Choice 2 (`./Threads 2`) — `callJoinInCatchBlock()`**
  - `createProblem()` throws, the `catch` calls `t.join()` and waits for `hello()` to finish.
  - Output order will typically be:
    ```
    Welcome to Concurrency in C++
    Hello from callJoinInCatchBlock!
    Hello from createProblem!
    Hello from hello!
    Bye from callJoinInCatchBlock!
    ========== Program ended cleanly ==========
    ```

- **Choice 3 (`./Threads 3`) — `callJoinInThreadGuard()`**
  - RAII ensures the thread is joined in the `ThreadGuard` destructor.
  - Output typically:
    ```
    Welcome to Concurrency in C++
    Hello from callJoinInThreadGuard!
    Hello from createProblem!
    Bye from callJoinInThreadGuard!
    Hello from hello!
    ========== Program ended cleanly ==========
    ```
  - Note: Because `tg` is destroyed at function exit, the `join()` may happen after the `Bye...` line but before returning to `main()`.

---

## Build instructions

### GNU/Linux (g++)
```bash
g++ -std=c++17 Threads.cpp -o Threads -pthread
```
(`-pthread` is required on many POSIX systems.)

### MinGW / Windows (g++)
```bash
g++ -std=c++17 Threads.cpp -o Threads.exe
```
(Depending on your MinGW distribution, `-pthread` may be unnecessary or produce warnings.)

### MSVC (Visual Studio Developer Command Prompt)
```bat
cl /EHsc /std:c++17 Threads.cpp
```

---

## Improvements & alternatives

### 1) Make the guard **own** the `std::thread` (`scoped_thread`)
A common, safer pattern is to take the `std::thread` **by value** and ensure it is joined in the destructor. That makes the wrapper own the thread and prevents lifetime bugs.

```cpp
// scoped_thread.h
#ifndef SCOPED_THREAD_H
#define SCOPED_THREAD_H

#include <thread>
#include <utility>

class scoped_thread {
    std::thread t_;
public:
    explicit scoped_thread(std::thread t) : t_(std::move(t)) {
        if (!t_.joinable())
            throw std::logic_error("No thread");
    }
    ~scoped_thread() {
        if (t_.joinable())
            t_.join();
    }
    scoped_thread(const scoped_thread&) = delete;
    scoped_thread& operator=(const scoped_thread&) = delete;

    // allow move
    scoped_thread(scoped_thread&&) = default;
    scoped_thread& operator=(scoped_thread&&) = default;
};

#endif // SCOPED_THREAD_H
```

Use it like:
```cpp
std::thread t(hello);
scoped_thread st(std::move(t));
```

### 2) Use `std::jthread` (C++20)
If you compile with C++20, prefer `std::jthread` — it automatically joins on destruction and supports cooperative cancellation via `stop_token`.

---

## Final notes (best practices)
- **Always** ensure a `std::thread` is either `join()`ed or `detach()`ed before it is destroyed. Otherwise the program will call `std::terminate()`.
- Prefer **RAII** (either `scoped_thread` owning wrapper, or `std::jthread`) to avoid manual `try`/`catch` bookkeeping.
- Avoid swallowing exceptions silently with `catch(...) {}` — at least log or rethrow after cleaning up resources.
- When using reference-based guards (like `ThreadGuard` above), be careful with object lifetimes: the `std::thread` must outlive the guard.



---

# `join()` vs `detach()` — Best Practices

In C++ concurrency, a `std::thread` must be either **joined** or **detached** before its destructor runs.  
If not, the runtime will call `std::terminate()`.  

This section explains the differences and when to use each.

---

## 1. Calling `detach()` immediately after launch

```cpp
std::thread t(hello);
t.detach();
```

- `detach()` makes the thread **run independently** (in the background).  
- The calling thread does not block.  
- After detaching, you cannot `join()` the thread anymore.  
- Good for fire-and-forget tasks (e.g., logging, async telemetry).

⚠️ **Be careful**: a detached thread must never access objects that might go out of scope, otherwise you get undefined behavior.

---

## 2. Why we cannot always call `join()` immediately

```cpp
std::thread t(do_work);
t.join();  // blocks until do_work finishes
```

- `join()` blocks until the thread finishes.  
- If you call `join()` right after creating each thread in a loop, they will execute **serially** instead of concurrently.  
- Often you want the main thread to keep doing other work, then later synchronize with the worker threads.

---

## 3. Ensuring `join()` happens eventually

If a `std::thread` object is destroyed while still **joinable**, the program calls `std::terminate()`.  
To avoid this, ensure that every thread is either joined or detached.  

### a) Manual join in a `catch`
```cpp
std::thread t(worker);
try {
    createProblem();
    t.join();
} catch(...) {
    if (t.joinable()) t.join();  // safe cleanup
}
```

### b) RAII with `ThreadGuard`
```cpp
std::thread t(worker);
ThreadGuard tg(t);  // joins automatically in destructor
createProblem();    // if exception, tg still joins
```

### c) Owning `scoped_thread`
```cpp
scoped_thread st(std::thread(worker));  // joins in destructor
```

### d) Modern C++20+ solution: `std::jthread`
```cpp
std::jthread t(worker);  // automatically joins in destructor
```

---

## ⚖️ Summary

- Use **`join()`** when you must synchronize with the thread before moving on.  
- Use **`detach()`** only for fire-and-forget tasks, and only when you know no dangling references will occur.  
- Never let a joinable thread go out of scope → it will cause `std::terminate()`.  
- Best practice: use **RAII wrappers** (`ThreadGuard`, `scoped_thread`, or `std::jthread`) to guarantee correct cleanup.

---


## 📖 Learn more about RAII
- RAII, or Resource Acquisition Is Initialization, is a programming idiom where the lifetime of a resource (like memory, a file handle, or a mutex) is tied to the lifetime of an object. In this pattern, the resource is acquired in an object's constructor and released in its destructor, automatically ensuring the resource is managed even if errors occur or the program flow is interrupted. This technique eliminates resource leaks and guarantees exception safety by leveraging object lifetime and scope exi
- For a deeper dive into the RAII (Resource Acquisition Is Initialization) idiom in C++, see:  
  [RAII on CppReference](https://en.cppreference.com/w/cpp/language/raii) (open source reference documentation)


