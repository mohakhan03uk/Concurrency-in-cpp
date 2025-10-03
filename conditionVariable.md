# Condition Variables in C++20

## 🔹 What is a condition variable (CV)?

A **condition variable** lets one thread **wait** until another thread
signals that something has happened.\
It's **event-driven** instead of **polling**.\
A condition variable is a synchronization primitive that allows one or more threads to wait until notified by another thread. They’re commonly used together with a mutex and a predicate (boolean condition).

------------------------------------------------------------------------
#### Example 
```
	#include <iostream>
	#include <thread>
	#include <mutex>
	#include <condition_variable>
	#include <chrono>

	std::mutex mtx;
	std::condition_variable cv;
	bool ready = false;  // shared state

	void worker(int id) {
		std::unique_lock<std::mutex> lock(mtx);

		// Wait until "ready == true"
		cv.wait(lock, [] { return ready; });

		// Once condition is met
		std::cout << "Worker " << id << " is processing\n";
	}

	int main() {
		std::thread t1(worker, 1);
		std::thread t2(worker, 2);

		std::this_thread::sleep_for(std::chrono::seconds(1));

		{
			std::lock_guard<std::mutex> lock(mtx);
			ready = true;  // update condition
		}
		cv.notify_all();  // wake up all waiting threads

		t1.join();
		t2.join();

		return 0;
	}

```
###### Key points:
- cv.wait(lock, predicate)
  - Atomically unlocks the mutex and suspends the thread until either:
    - It’s notified via notify_one() or notify_all()
    - The predicate evaluates to true
- notify_one() wakes up a single waiting thread.
- notify_all() wakes up all waiting threads.
- ***Always use a loop/predicate to avoid spurious wakeups.***
## 🔹 Why not just use `sleep()` or a `while` loop?

### 1. Polling with `while + sleep` (inefficient)

``` cpp
while (!ready) {
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
}
```

-   **CPU waste**: The thread wakes up every 10ms even if nothing
    changed.\
-   **Latency**: The check only happens every 10ms, so events are not
    detected instantly.\
-   **Hard to tune**: If you reduce sleep (e.g. 1ms), CPU usage spikes;
    if you increase it, responsiveness drops.

### 2. Busy-wait (just `while`)

``` cpp
while (!ready) {
    // keeps checking
}
```

-   **Worst case**: 100% CPU usage doing nothing.

### 3. Using a condition variable

``` cpp
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

-   **No CPU waste**: The thread sleeps efficiently inside the OS until
    notified.\
-   **Instant reaction**: Wakes up immediately when
    `notify_one`/`notify_all` is called.\
-   **Safe**: Works with a predicate, so spurious wakeups don't break
    logic.

------------------------------------------------------------------------

## 🔹 Alternatives to condition variables

1.  **Futures & Promises (`std::future`, `std::promise`)**

    -   You can wait for a value from another thread:

    ``` cpp
    std::promise<void> p;
    std::future<void> f = p.get_future();
    std::thread t([&p]{ p.set_value(); });
    f.wait();  // blocks until set_value() is called
    ```

    - Simpler for **one-time events**.\
    - Less flexible for repeated signaling (producer-consumer).

2.  **Atomics + busy-wait with backoff**

    -   Use `std::atomic<bool>` with a `while` loop and`std::this_thread::yield()`.
    - Works without mutex/condvar.
    - Still wastes CPU, less efficient than CV.

3.  **Semaphores (`std::counting_semaphore` in C++20)**

    -   Similar to CV, but keeps a count of signals.

    ``` cpp
    std::counting_semaphore<1> sem(0);
    sem.release();  // signal
    sem.acquire();  // wait
    ```

    - Often simpler than CVs for producer-consumer.
    - Less flexible when you need a complex condition (like multiple flags).
------------------------------------------------------------------------

# Condition Variable vs Semaphore in C++20

## Condition Variable Example (Producer-Consumer)

``` cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> q;
bool done = false;

void producer() {
    for (int i = 1; i <= 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            q.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        cv.notify_one(); // wake up one consumer
    }

    {
        std::lock_guard<std::mutex> lock(mtx);
        done = true;
    }
    cv.notify_all(); // wake up all waiting consumers
}

void consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !q.empty() || done; });

        if (!q.empty()) {
            int val = q.front();
            q.pop();
            lock.unlock();
            std::cout << "Consumed: " << val << "\n";
        } else if (done) {
            break;
        }
    }
}

int main() {
    std::thread p(producer);
    std::thread c1(consumer), c2(consumer);

    p.join();
    c1.join();
    c2.join();

    return 0;
}
```

------------------------------------------------------------------------

## Semaphore Example (Producer-Consumer)

``` cpp
#include <iostream>
#include <thread>
#include <queue>
#include <semaphore> // C++20

std::queue<int> q;
std::binary_semaphore sem_empty(1); // allow push
std::counting_semaphore<10> sem_full(0); // count produced items
bool done = false;

void producer() {
    for (int i = 1; i <= 5; ++i) {
        sem_empty.acquire(); // wait until space is available
        q.push(i);
        std::cout << "Produced: " << i << "\n";
        sem_full.release(); // signal item is available
    }
    done = true;
    sem_full.release(); // unblock consumers
}

void consumer() {
    while (true) {
        sem_full.acquire(); // wait until item is available

        if (!q.empty()) {
            int val = q.front();
            q.pop();
            std::cout << "Consumed: " << val << "\n";
            sem_empty.release(); // allow producer to push again
        } else if (done) {
            break;
        }
    }
}

int main() {
    std::thread p(producer);
    std::thread c1(consumer), c2(consumer);

    p.join();
    c1.join();
    c2.join();

    return 0;
}
```
-   **Semaphore** → Better for controlling *number of available
    resources*.
