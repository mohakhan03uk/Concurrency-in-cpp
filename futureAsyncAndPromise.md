# `std::async`, `std::future` and `std::promise` in C++

This document explains the basics of `std::future`, how it is used with `std::async` and `std::promise`,
and then provides a deep dive into the difference between passing a `std::promise` to a worker thread
via `std::move` (transfer of ownership) versus `std::ref` (reference).

---

## What is `std::future`?
`std::future` is a C++ Standard Library feature that represents **a value that will become available in the future**—usually the result of an asynchronous computation.

- Declared in `<future>` (since C++11).
- Works with `std::async`, `std::promise`, and `std::packaged_task`.
- Used to retrieve results from asynchronous operations.

### Key Methods
- `get()` → retrieve the result (blocks until ready).
- `wait()` → wait until the result is available (does not retrieve).
- `wait_for(duration)` → wait with timeout.
- `wait_until(time_point)` → wait until a specific time.

## What is `std::async`?
std::async in C++ is a utility from the `<future>` header that allows you to run a function asynchronously (in a separate thread or lazily deferred) and get its result via a std::future.\
It’s often used when you want concurrency but don’t want to manage threads manually.
```cpp
    #include <future>
    // general form
    std::future<ReturnType> f = std::async(launch_policy, function, args...);
```
- launch_policy (optional):
    - std::launch::async → run in a new thread immediately.
    - std::launch::deferred → run lazily, only when future.get() or future.wait() is called.
    - If omitted, implementation may choose either.
- function: Callable (function pointer, lambda, functor, etc.).
- args...: Arguments forwarded to the callable.

## What is `std::promise` ?
std::promise in C++ is a synchronization mechanism from the Concurrency (Futures and Promises) library (`<future>`).\
It allows one thread (the producer) to set a value (or exception), and another thread (the consumer) to retrieve it asynchronously using a corresponding std::future.
#### Key Points about std::promise
- Declared in `<future>`.
- A `std::promise<T>` is linked to a `std::future<T>`.
- You use promise.set_value() (or set_exception()) in one thread, and the future’s get() in another.
- The promise can only be fulfilled once (either value or exception).
- Useful for communication between threads.

---

## Example 1: Using `std::async` + `std::launch::async`  --> Asynchronous computation

```cpp
#include <iostream>
#include <future>
#include <thread>

int compute_square(int x) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return x * x;
}

int main() {
    // Run asynchronously
    std::future<int> result = std::async(std::launch::async, compute_square, 5);

    std::cout << "Doing other work...\n";

    // Block until result is ready
    std::cout << "Result: " << result.get() << "\n";
}
```
#### Output
`Doing other work...`\
`Result: 25`

**Explanation:**
- `std::async` launches `compute_square(5)` in a new thread.
- `result` is a `std::future<int>` holding the eventual return value. the compute_square() is called in different thread, you can put thread id print in compute_square() and main() to understad more.
- `result.get()` blocks until the function finishes, then returns 25.

---

## Example 2: Using `std::promise` + `std::future`

```cpp
#include <iostream>
#include <future>
#include <thread>

void worker(std::promise<int> p) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    p.set_value(42); // set result
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t(worker, std::move(p));

    std::cout << "Waiting for result...\n";
    std::cout << "Result: " << f.get() << "\n"; // waits until set_value is called

    t.join();
}
```
#### Output
`Waiting for result...`\
`Result: 42`

#### Explanation (Step by Step)
- `std::promise<int>` p;
  - A promise object is created. It will eventually provide an int result.
- std::future<int> f = p.get_future();
  - A future is tied to the promise.
  - f will receive the value that p.set_value() provides.
- `std::thread t(worker, std::move(p));`
  - A new thread is started and given ownership of the promise (`std::move`).
  - Each `std::promise` should have exactly one owner responsible for setting its value, so std::move ensures only the worker has it.
  - The worker thread sleeps for 2 seconds, then calls p.set_value(42).
- Meanwhile in main():
  - It prints "Waiting for result...".
  - Then calls f.get().
    - This call blocks until the worker thread sets the promise value.
- After 2 seconds, the worker thread sets the value (42).
  - The future (f) becomes ready.
  - f.get() returns 42.
- "Result: 42" is printed.
- t.join() ensures the worker thread finishes cleanly.
---

## Example 2: Using `std::async` + `std::launch::deferred`  --> Deferred execution

```cpp
#include <iostream>
#include <future>

int multiply(int a, int b) {
    std::cout << "multiply executed!\n";
    return a * b;
}

int main() {
    // Task won't run until .get() is called
    std::future<int> result = std::async(std::launch::deferred, multiply, 6, 7);

    std::cout << "Task not started yet.\n";

    // Now it runs
    std::cout << "Result: " << result.get() << "\n";
}

```
###  Key Notes
- `std::async(std::launch::deferred, multiply, 6, 7)` creates a deferred task → nothing runs yet.
- Program prints:  `Task not started yet.`
- `result.get()` is called → only then `multiply(6,7)` executes.
  - It first prints:  `multiply executed!`
- Then returns 42, which is printed:  `Result: 42`
- Final Output:
  ```cpp
  Task not started yet.
  multiply executed!
  Result: 42

  ```
#### More
-  Return values are obtained via .get(), which blocks until the task is done.
-  If the function `multiply` throws an exception, .get() will rethrow it.
-  If you don’t specify a launch policy, the implementation decides (it might run async or deferred).
-  ***`std::async` is simpler than manually creating std::thread + std::promise.***

---
---

## Pass by Reference vs `std::move`

### Version 1: Ownership transfer with `std::move`

```cpp
#include <iostream>
#include <future>
#include <thread>

void worker(std::promise<int> p) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    p.set_value(42); // Worker sets the value
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t(worker, std::move(p)); // Ownership transferred to worker

    std::cout << "Main waiting...\n";
    std::cout << "Result: " << f.get() << "\n";

    t.join();
}
```

**What happens:**
- Worker is the **sole owner** of `p`.
- `main` cannot use `p` after `std::move`.
- Clean hand-off, safe, predictable → **Result: 42**.

---

### Version 2: Reference passing with `std::ref`

```cpp
#include <iostream>
#include <future>
#include <thread>

void worker(std::promise<int>& p) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    p.set_value(42); // Worker sets the value
}

int main() {
    std::promise<int> p;
    std::future<int> f = p.get_future();

    std::thread t(worker, std::ref(p)); // Worker borrows the same promise

    // If main ALSO sets the value → ERROR
    // p.set_value(100);  // uncomment this and you'll get exception

    std::cout << "Main waiting...\n";
    std::cout << "Result: " << f.get() << "\n";

    t.join();
}
```

**What happens:**
- `main` and the worker both refer to the **same promise** object.
- If only the worker calls `set_value()`, works fine → **Result: 42**.
- If `main` also calls `set_value()`, you’ll get:

```
terminate called after throwing an instance of 'std::future_error'
  what():  Promise already satisfied
```

because a promise’s result can only be set **once**.

---

## Takeaways

- **`std::move`** → transfers ownership to worker (only worker sets the value). → *safe & idiomatic*.
- **`std::ref`** → shared object → *possible misuse; exception if set twice*.
- Passing by reference works, but it blurs ownership.
- `std::promise` is **non-copyable** but **movable**.
- Use `std::move` when you want to clearly hand off responsibility to another thread.
- Moving makes it clear: ******“I’m giving this promise to the worker, and I won’t touch it again.”*******

---
