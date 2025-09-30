# 🧵 Usefull Fuctions while working with threads in C++

| Function / Utility | Header | Description | Mini Example |
|---------------------|---------|-------------|--------------|
| **Constructor** `std::thread t(func, args...)` | `<thread>` | Creates a new thread running `func`. | <pre>std::thread t([](){ std::cout << "Worker\n"; });<br>t.join();</pre> |
| **`join()`** | `<thread>` | Waits for the thread to finish. | <pre>std::thread t([](){ std::cout << "Joining\n"; });<br>t.join();</pre> |
| **`detach()`** | `<thread>` | Runs thread in background, cannot be joined later. | <pre>std::thread t([](){ std::cout << "Detached\n"; });<br>t.detach();</pre> |
| **`joinable()`** | `<thread>` | Checks if thread can be joined. | <pre>std::thread t([](){});<br>if (t.joinable()) t.join();</pre> |
| **`get_id()`** | `<thread>` | Returns thread’s unique ID. | <pre>td::thread t([](){});<br>std::cout << t.get_id();<br>t.join();</pre> |
| **`std::this_thread::get_id()`** | `<thread>` | Gets ID of current thread. | <pre>std::cout << std::this_thread::get_id();</pre> |
| **`std::thread::hardware_concurrency()`** | `<thread>` | Returns number of CPU-supported threads. | <pre>unsigned n = std::thread::hardware_concurrency();<br>std::cout << n;</pre> |
| **`std::this_thread::sleep_for(duration)`** | `<thread>`, `<chrono>` | Suspends thread for `duration`. | <pre>using namespace std::chrono_literals;<br>std::this_thread::sleep_for(1s);</pre> |
| **`std::this_thread::sleep_until(time_point)`** | `<thread>`, `<chrono>` | Suspends until given time point. | <pre>using clk = std::chrono::system_clock;<br>std::this_thread::sleep_until(clk::now()+1s);</pre> |
| **`std::this_thread::yield()`** | `<thread>` | Gives up CPU voluntarily. | <pre>std::this_thread::yield();</pre> |

---

## ✅ Common Headers
- `<thread>` → `std::thread`, `join`, `detach`, `joinable`, `get_id`, `this_thread::*`, `hardware_concurrency`  
- `<chrono>` → `sleep_for`, `sleep_until`  

---

## 📌 Example Program (Demonstrates Everything)

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void worker(int id) {
    std::cout << "Worker " << id << " running on thread "
              << std::this_thread::get_id() << "\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}

int main() {
    std::cout << "Main thread ID: " << std::this_thread::get_id() << "\n";
    std::cout << "Hardware concurrency: "
              << std::thread::hardware_concurrency() << "\n";

    std::thread t1(worker, 1);
    std::thread t2(worker, 2);

    if (t1.joinable()) t1.join();
    if (t2.joinable()) t2.join();

    std::cout << "Main thread yielding...\n";
    std::this_thread::yield();

    std::cout << "Sleeping main thread for 1s...\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));

    std::cout << "Done.\n";
    return 0;
}
