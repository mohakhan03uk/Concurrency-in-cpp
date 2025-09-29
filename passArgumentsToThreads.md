# Passing Arguments to `std::thread` in C++

The `std::thread` constructor has the following signature:

```cpp
template <class Function, class... Args>
explicit thread(Function&& f, Args&&... args);
```

- `f` → The callable (function, functor, lambda, member function pointer).
- `args...` → Arguments to pass to the callable.
- Arguments are **copied or moved** into the new thread context.

---

## 1. Pass by Value (default)

Arguments are copied or moved.

```cpp
#include <iostream>
#include <thread>

void func(int x, std::string s) {
    std::cout << "x=" << x << ", s=" << s << "\n";
}

int main() {
    std::thread t(func, 42, std::string("Hello"));
    t.join();
}
```

**Output:**
```
x=42, s=Hello
```

---

## 2. Pass by Reference

Use `std::ref` or `std::cref` to preserve references.

```cpp
#include <iostream>
#include <thread>
#include <functional>

void func(int& x) { x += 10; }

int main() {
    int value = 5;
    std::thread t(func, std::ref(value)); // reference
    t.join();
    std::cout << value << "\n"; // prints 15
}
```

**Output:**
```
15
```

If you forget `std::ref`:

```cpp
#include <iostream>
#include <thread>

void func(int& x) { x += 10; }

int main() {
    int value = 5;
    std::thread t(func, value); // ❌ passes by value, doesn't modify original
    t.join();
    std::cout << value << "\n"; // still 5
}
```

**Output:**
```
5
```

---

## 3. Using Lambdas

Lambdas are the most modern and flexible way.

```cpp
#include <iostream>
#include <thread>

int main() {
    int a = 10;
    std::string s = "World";

    std::thread t([=]() { // capture by value
        std::cout << a << " " << s << "\n";
    });
    t.join();

    std::thread t2([&]() { // capture by reference
        a += 5;
    });
    t2.join();
    std::cout << "a=" << a << "\n";
}
```

**Output:**
```
10 World
a=15
```

---

## 4. Member Functions

Pass the member function pointer and the object.

```cpp
#include <iostream>
#include <thread>

class Worker {
public:
    void doWork(int x) {
        std::cout << "Work: " << x << "\n";
    }
};

int main() {
    Worker w;
    std::thread t(&Worker::doWork, &w, 42);
    t.join();
}
```

**Output:**
```
Work: 42
```

---

## 5. Functor Objects

```cpp
#include <iostream>
#include <thread>

struct Functor {
    void operator()(int x) { std::cout << "x=" << x << "\n"; }
};

int main() {
    Functor f;
    std::thread t(f, 100);
    t.join();
}
```

**Output:**
```
x=100
```

---

## 6. Move-Only Parameters

Some C++ types **cannot be copied**, only **moved**.  
Examples include:
- `std::unique_ptr<T>` (owning smart pointer)  
- `std::thread` itself  
- Certain file/socket handles  

These types delete their copy constructor but allow move construction.  
Since `std::thread` normally copies arguments, you must explicitly use `std::move` for move-only objects.

---

### Example 1: `std::unique_ptr`

```cpp
#include <iostream>
#include <thread>
#include <memory>

void func(std::unique_ptr<int> p) {
    std::cout << "Value=" << *p << "\n";
}

int main() {
    auto ptr = std::make_unique<int>(42);

    // Must move ownership into the thread
    std::thread t(func, std::move(ptr));
    t.join();

    // ptr is now nullptr because ownership moved to the thread
    if (!ptr) {
        std::cout << "ptr is empty in main\n";
    }
}
```
**Key Points for Move-Only Types**

- Move-only types cannot be copied into threads.
- Always use std::move when passing them to std::thread.
- Ownership transfer happens: The original variable becomes “empty” (like nullptr) after the move.
- Thread-safety by design : Only one thread can own a move-only resource at any time.

---
---

## 7. Using `std::bind`

### Why `std::bind`?

`std::thread` expects a **callable object** (function, functor, or lambda).  
Sometimes you need to:
1. **Fix certain arguments** in advance.  
2. **Rearrange arguments** using placeholders.  
3. **Bind member functions** to a specific object instance.  

`std::bind` creates a new callable object with these arguments bound, which can then be passed directly to a thread.

---

### Example 1: Binding free function with fixed args

```cpp
#include <iostream>
#include <thread>
#include <functional>

void print_sum(int a, int b) {
    std::cout << "Sum = " << (a + b) << "\n";
}

int main() {
    // bind fixes arguments to 10 and 20
    auto boundFunc = std::bind(print_sum, 10, 20);

    std::thread t(boundFunc); // no args needed
    t.join();
}
```
- Here, std::bind creates a no-argument callable that always calls print_sum(10, 20).

### Example 2: Using placeholders 
```
#include <iostream>
#include <thread>
#include <functional>

void multiply(int x, int y) {
    std::cout << "Product = " << (x * y) << "\n";
}

int main() {
    // First param fixed = 5, second param left open (_1)
    auto boundFunc = std::bind(multiply, 5, std::placeholders::_1);

    std::thread t(boundFunc, 10); // _1 = 10
    t.join();
}
```
- Here, std::bind makes a callable that always multiplies by 5, while the second parameter is provided later when launching the thread.
---

### Example 3: Binding member function to an object
```
#include <iostream>
#include <thread>
#include <functional>

class Worker {
public:
    void doWork(int times) {
        for (int i = 0; i < times; ++i) {
            std::cout << "Working... " << i+1 << "\n";
        }
    }
};

int main() {
    Worker w;

    // Bind member function to object 'w' with times=3
    auto boundFunc = std::bind(&Worker::doWork, &w, 3);

    std::thread t(boundFunc);
    t.join();
}
```
- std::bind allows you to attach the member function doWork to a specific object (w) and pre-bind its arguments.

**Key Points about std::bind**
- Creates a callable object with pre-set arguments
- Supports placeholders (_1, _2, etc.) for arguments supplied later.
- Works well with member functions that require an object.
- Today, many developers prefer lambdas for simplicity: std::thread t([&] { multiply(5, 10); });
- But std::bind is still useful when you need reusability or partial application.

  
# ✅ Summary

| Method            | Syntax                                       | Notes |
|-------------------|----------------------------------------------|-------|
| Pass by value     | `std::thread t(func, 10, "hi");`            | Default copy/move |
| Pass by reference | `std::thread t(func, std::ref(x));`         | Use `std::ref`/`std::cref` |
| Lambda            | `std::thread t([&](){ ... });`              | Modern, flexible |
| Member function   | `std::thread t(&Class::method, &obj, args);`| Pass object pointer/ref |
| Functor           | `std::thread t(Functor(), arg);`            | Works like lambdas |
| Move-only         | `std::thread t(func, std::move(ptr));`      | Transfers ownership |
| std::bind         | `std::thread t(std::bind(func, arg1, arg2));` | Old style, less clean |

---

# 🚀 Recommendation

- Prefer **lambdas** for clarity and modern style.  
- Use `std::ref` when you really need references.  
- Avoid `std::bind` in new code (use lambdas instead).  
