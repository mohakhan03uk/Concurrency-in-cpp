# HelloWorld with std::thread()

This file demonstrates using `std::thread` in C++ with **join** and **without join**.

## Example 1: Without `join()` (Incorrect)

```cpp
#include <iostream>
#include <thread>

void hello() {
    std::cout << "Hello from thread!" << std::endl;
}

int main() {
    std::cout << "Welcome to Concurrency in C++" << std::endl;

    std::thread t(hello);   // start the thread

    // ERROR: no join or detach!
    // when main exits, t is destroyed while still joinable
    // this causes std::terminate() to be called

    return 0;
}
```

### Expected Behavior

- Program prints:
  ```
  Welcome to Concurrency in C++
  terminate called without an active exception
  Hello from thread!
  ```

- The `std::thread` destructor calls `std::terminate()` because `t` was still joinable.

---

## Example 2: With `join()` (Correct Way)

```cpp
#include <iostream>
#include <thread>

void hello() {
    std::cout << "Hello from thread!" << std::endl;
}

int main() {
    std::cout << "Welcome to Concurrency in C++" << std::endl;

    std::thread t(hello);   // start the thread

    if (t.joinable()) {
        t.join();   // wait for the thread to finish
    }

    return 0;
}
```

### Compilation (Windows with MinGW-w64)

```bash
g++ -std=c++17 main.cpp -o hello.exe
.\hello.exe
```

### Expected Output

```
Welcome to Concurrency in C++
Hello from thread!
```

---

---
## Key Rule
A `std::thread` **must be either joined or detached** before it is destroyed.  
If not, the program crashes with `std::terminate()`.
