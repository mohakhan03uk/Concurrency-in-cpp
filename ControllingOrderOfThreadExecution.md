# 🧵 Structured Thread Execution in C++

## 📌 Problem Statement  
In multithreaded programming, threads often run **concurrently**, producing **non-deterministic output**.  
But in some cases, we may want to **order thread execution** to maintain **predictability and control**.  

The goal of this program is to:  
1. Create multiple threads (`A`, `B`, and `C`).  
2. Ensure they execute in a **specific order**:  
   - Thread **A** runs and completes.  
   - Thread **B** runs and completes.  
   - Inside Thread **B**, create and run Thread **C**, which must complete **before B finishes**.  
3. Demonstrate the use of `std::thread`, `joinable()`, and `join()` to enforce sequential execution.  

---

## 🎯 Learning Objectives  
- Learn how to create threads using `std::thread`.  
- Understand `joinable()` checks for safe thread joining.  
- Use `join()` to enforce **structured execution order**.  
- Build a nested thread model where a parent thread manages its own child threads.  

---

## 📝 Code Overview  

```cpp
#include <iostream>
#include <thread>

void helloFromA() {
    std::cout << "Hello from thread A!\n";
}

void helloFromC() {
    std::cout << "Hello from thread C!\n";
}

void helloFromB() {
    std::cout << "Hello from thread B!\n";

    std::thread tC(helloFromC);   // Start thread C
    if (tC.joinable())
        tC.join();                // Ensure C finishes before B ends
}

int main() {
    std::cout << "Welcome to Concurrecy in cpp" << std::endl;

    std::thread tA(helloFromA);   // Start thread A
    if (tA.joinable())
        tA.join();                // Wait for A to finish

    std::thread tB(helloFromB);   // Start thread B
    if (tB.joinable())
        tB.join();                // Wait for B (and its child C) to finish

    return 0;
}
```

---

## 🖥️ Sample Output  

```
Welcome to Concurrecy in cpp
Hello from thread A!
Hello from thread B!
Hello from thread C!
```

---

## ✅ Key Takeaways  
- Threads can be **structured** to follow a defined execution order.  
- `join()` is the key mechanism to block a parent thread until a child thread finishes.  
- Even though threads usually introduce concurrency, you can use them in a **controlled sequential manner** when needed.

