# Stage 3: The System Protector (Bounded Queue & Backpressure)

## Overview
An unbounded queue will eventually consume all system memory (OOM crash) if producers generate data faster than consumers can process it. A Bounded Queue introduces **Backpressure**—putting producers to sleep when the queue reaches a strict capacity limit. 

This implementation also introduces **Graceful Shutdown** mechanics, allowing the queue to be flushed and safely terminating blocked threads during a system SIGTERM.

## Functional Requirements
* **Capacity Limit:** The queue must not exceed `N` items.
* **Producer Blocking:** If the queue is full, `enqueue` must block until space is freed.
* **Consumer Blocking:** If the queue is empty, `dequeue` must block until data arrives.
* **Graceful Termination:** A `stopAll` mechanism must wake up all sleeping threads and safely reject new operations.

## Architecture: The Out-Parameter Pattern vs. RVO

In Stage 2, we returned values using Return Value Optimization (RVO). Here, we demonstrate the **Out-Parameter Pattern**, heavily used in High-Frequency Trading and game engines.

| Feature | Return By Value (RVO) `T dequeue()` | Out-Parameter `bool dequeue(T& out)` |
| :--- | :--- | :--- |
| **Error Handling** | Requires returning `std::optional<T>` or throwing exceptions. | Returns a `bool` (Success/Fail) or Enum. Cleanest way to handle shutdowns. |
| **Memory Control** | The function creates the object and gives it to the caller. | The Caller pre-allocates the memory exactly where they want it. |
| **Performance** | Extremely fast (compiler optimizes away copies). | Extremely fast (direct memory move into caller's pre-allocated space). |

## Implementation: C++17

```cpp
#include <condition_variable>
#include <queue>
#include <mutex>
#include <iostream>

template <typename T>
class ThreadSafeBoundedQueue {
private:
    mutable std::mutex mtx;
    std::condition_variable not_empty;
    std::condition_variable not_full;
    
    size_t capacity;
    std::queue<T> q;
    bool isFinished;

public:
    // 'explicit' prevents accidental type conversions (e.g., Queue q = 10)
    // Initializer list sets up primitives before constructor body
    explicit ThreadSafeBoundedQueue(size_t n) : capacity(n), isFinished(false) {} 
    
    // Returns true if successfully enqueued, false if queue is shutting down
    bool enqueue(const T& task) {
        std::unique_lock<std::mutex> lock(mtx);
        
        // Sleep if queue is full, UNLESS we are shutting down
        not_full.wait(lock, [this]() { 
            return (q.size() < capacity) || isFinished; 
        });

        if (isFinished) {
            return false; // Safely reject without throwing expensive exceptions
        }

        q.push(task);
        
        // Wake up one sleeping consumer
        not_empty.notify_one(); 
        return true;
    }

    // Returns true if successfully dequeued, false if queue is closed & empty
    bool dequeue(T& out_item) {
        std::unique_lock<std::mutex> lock(mtx);
    
        // Sleep if queue is empty, UNLESS we are shutting down
        not_empty.wait(lock, [this]() { 
            return !q.empty() || isFinished; 
        });

        // If shutdown was called AND the queue is completely drained, exit safely
        if (isFinished && q.empty()) {
            return false;
        }

        // Steal the memory pointers from the queue item into the caller's reference
        out_item = std::move(q.front()); 
        q.pop();
        
        // Wake up one sleeping producer
        not_full.notify_one(); 
        return true;
    }

    // Gracefully wake all threads and close the queue
    void stopAll() {
        std::lock_guard<std::mutex> lock(mtx);
        if (isFinished) return;
        
        isFinished = true;
        
        // Wake up everyone. Producers will return false, Consumers will drain remaining items.
        not_empty.notify_all();
        not_full.notify_all();
    }
    
    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx);
        return q.size();
    }
};
