# Understanding `std::packaged_task` in C++

The class template `std::packaged_task` wraps any callable target/object (function, lambda, or functor) that allows you to :
- execute it asynchronously
- and retrieve the result via a `std::future`.
- It's return value or exception thrown, is stored in a shared state which can be accessed through `std::future` objects.\
Essentially, it connects a callable and a future, so you can retrieve the result once the task finishes.

---

## Basic Syntax (Synchronous Example)

```cpp
#include <future>
#include <iostream>

int add(int a, int b) {
    return a + b;
}

int main() {
    // 1. Create a packaged_task wrapping a function
    std::packaged_task<int(int, int)> task(add);

    // 2. Get the future associated with this task
    std::future<int> result = task.get_future();

    // 3. Execute the task
    task(2, 3);

    // 4. Get the result from the future
    std::cout << "Result: " << result.get() << std::endl; // prints 5
}
```

---

## Key Points

1. **Template parameter**:
```cpp
std::packaged_task<ReturnType(ArgTypes...)>
```
You must specify the function signature.

2. **Callable**: Can be a function, lambda, or functor.

3. **Future**: Only one `std::future` can be obtained per `std::packaged_task`.

4. **Move-only**: `packaged_task` is move-only and cannot be copied.

---
##### Step-by-Step Execution

- Create packaged_task :
    `std::packaged_task<int(int, int)> task(add);`
  - Wraps the add function.
  - Stores the callable and allows a future to retrieve the result.
- Get the future : `std::future<int> result = task.get_future();`
  - result will eventually hold the return value of add(2, 3).
- Execute the task   : `task(2, 3);`
  - Calls add(2, 3).
  - The return value is stored internally in the future.
- Retrieve the result:  `std::cout << "Result: " << result.get() << std::endl;`
  - Waits for the task to complete (if it hasn’t already).
  - Gets the result (5 in this case).


## Threaded Example (Asynchronous Execution)

```cpp
#include <iostream>
#include <future>
#include <thread>

int add(int a, int b) {
    std::cout << "Task running in thread id: " << std::this_thread::get_id() << std::endl;
    return a + b;
}

int main() {
    std::packaged_task<int(int,int)> task(add);
    std::future<int> result = task.get_future();

    std::thread t(std::move(task), 10, 20);
    std::cout << "Main thread id: " << std::this_thread::get_id() << std::endl;

    t.join();
    std::cout << "Result from future: " << result.get() << std::endl;
}
```

### Expected Output
```
Main thread id: 140706969179584
Task running in thread id: 140706960786880
Result from future: 30
```
*Note: Thread IDs may vary.*

### Step-by-Step Execution
1. **Create `packaged_task`**: Wraps the `add` function.
2. **Get `future`**: Holds the return value for asynchronous execution.
3. **Run in separate thread**: `std::thread` executes the task asynchronously.
4. **Main thread continues**: Can do other work while task runs.
5. **Join thread**: Waits for task completion.
6. **Retrieve result**: `future.get()` returns the computed value (30).

---

## Why Use `std::packaged_task`?

- Decouple task execution from result retrieval.
- Combine with thread pools for asynchronous computation.
- Useful for building futures-based concurrency frameworks.
- If you have a thread pool, you can store many packaged_task objects in a queue and let worker threads execute them. This is hard to do with just functions because you need a way to retrieve each task's result safely.
- Encapsulates the callable and allows us to store it in a queue.
- Automatically provides a future to retrieve the result asynchronously.

---

#### Useful in Thread Pools
If you have a thread pool, you can store many packaged_task objects in a queue and let worker threads execute them. This is hard to do with just functions because you need a way to retrieve each task's result safely.
```cpp
    std::queue<std::packaged_task<int()>> taskQueue;
    std::vector<std::future<int>> results;
    
    // Create tasks
    for (int i=0; i<10; i++) {
        std::packaged_task<int()> task([i]{ return i*i; });
        results.push_back(task.get_future());
        taskQueue.push(std::move(task));
    }
    
    // Worker threads execute tasks
    while (!taskQueue.empty()) {
        auto t = std::move(taskQueue.front());
        taskQueue.pop();
        t(); // execute
    }
    
    // Collect results
    for (auto& r : results) std::cout << r.get() << " ";
```
Here, packaged_task handles the storage, execution, and future retrieval all in one object. Without it, you'd need to manually synchronize everything.
----

#### Integration with std::async
std::async internally uses packaged_task to run a function asynchronously and return a future. So, packaged_task is the building block of modern C++ async programming.

***`std::packaged_task`*** is useful because it:
-  Separates execution from result retrieval.
-  Works well with threads and thread pools.
-  Provides a future to safely get the result later.
-  Is a building block for std::async and more complex concurrent frameworks.

## Summary Diagram
```
[Callable] ---> [std::packaged_task] ---> [std::future] ---> get() retrieves result
```
---
## Thread Pool Example 
```cpp
      #include <iostream>
      #include <vector>
      #include <queue>
      #include <thread>
      #include <future>
      #include <functional>
      #include <mutex>
      #include <condition_variable>
      
      class ThreadPool {
      public:
          ThreadPool(size_t numThreads);
          ~ThreadPool();
      
          // Submit a task to the pool
          template <typename Func, typename... Args>
          auto submit(Func&& f, Args&&... args) -> std::future<decltype(f(args...))>;
      
      private:
          std::vector<std::thread> workers;
          std::queue<std::packaged_task<void()>> tasks;
          std::mutex queueMutex;
          std::condition_variable condition;
          bool stop = false;
      
          void workerThread();
      };
      
      // Constructor: start threads
      ThreadPool::ThreadPool(size_t numThreads) {
          for (size_t i = 0; i < numThreads; ++i) {
              workers.emplace_back([this]{ this->workerThread(); });
          }
      }
      
      // Destructor: stop all threads
      ThreadPool::~ThreadPool() {
          {
              std::unique_lock<std::mutex> lock(queueMutex);
              stop = true;
          }
          condition.notify_all();
          for (auto& thread : workers) {
              thread.join();
          }
      }
      
      // Worker thread function
      void ThreadPool::workerThread() {
          while (true) {
              std::packaged_task<void()> task;
              {
                  std::unique_lock<std::mutex> lock(queueMutex);
                  condition.wait(lock, [this]{ return stop || !tasks.empty(); });
                  if (stop && tasks.empty()) return;
                  task = std::move(tasks.front());
                  tasks.pop();
              }
              task(); // Execute task
          }
      }
      
      // Submit function
      template <typename Func, typename... Args>
      auto ThreadPool::submit(Func&& f, Args&&... args) -> std::future<decltype(f(args...))> {
          using RetType = decltype(f(args...));
      
          auto task = std::make_shared<std::packaged_task<RetType()>>(
              std::bind(std::forward<Func>(f), std::forward<Args>(args)...)
          );
      
          std::future<RetType> fut = task->get_future();
      
          {
              std::unique_lock<std::mutex> lock(queueMutex);
              tasks.emplace([task]{ (*task)(); }); // wrap in void() task
          }
          condition.notify_one();
      
          return fut;
      }
      
      // --- Example Usage ---
      int square(int x) {
          return x * x;
      }
      
      int main() {
          ThreadPool pool(3); // 3 worker threads
      
          std::vector<std::future<int>> results;
      
          // Submit tasks
          for (int i = 1; i <= 10; ++i) {
              results.push_back(pool.submit(square, i));
          }
      
          // Collect results
          for (auto& fut : results) {
              std::cout << fut.get() << " "; // 1 4 9 16 25 36 49 64 81 100
          }
          std::cout << std::endl;
      
          return 0;
      }
```
#### Explanation
- ThreadPool class:
  - Holds a vector of worker threads.
  - Has a queue of packaged_task<void()> to store tasks.
  - Uses a mutex + condition variable for synchronization.
- Submit tasks:
    - Wrap your callable in std::packaged_task.
    - Store in queue.
    - Notify a worker thread to execute it.
- Worker threads:
    - Wait for tasks.
    - Execute tasks asynchronously.
    - `future` allows retrieval of results later.
- Main thread:
    - Can submit multiple tasks without blocking.
    - Collect results when needed using future.get().
