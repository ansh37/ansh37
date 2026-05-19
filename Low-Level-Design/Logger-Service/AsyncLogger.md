# Asynchronous Logger Service

## Overview
A production-grade Logging Service (often asked by DocuSign, Uber, or Amazon) tests a candidate's understanding of **Non-Blocking I/O, the Producer-Consumer pattern, and Extensibility**. If a logger blocks the main application thread to write to a hard drive, it will throttle the entire distributed system. This design guarantees ultra-low latency for application threads by offloading the heavy I/O to a background worker.

## 1. Scope Boundary
### Functional Requirements (Core)
* **Varying Severities:** Support standard log levels (DEBUG, INFO, WARN, ERROR, FATAL).
* **Pluggable Sinks:** The logger must write to multiple destinations simultaneously (Console, File, Network/DB) without modifying the core engine.
* **Thread Tracking:** Every log must capture the `thread_id` and timestamp for tracing interleaved execution paths.

### Non-Functional Requirements (NFRs)
* **Thread-Safe & Non-Blocking:** Application threads calling `logger.log()` must return immediately. They should never wait for disk I/O.
* **Graceful Shutdown (Zero Data Loss):** If the application crashes or exits, the logger must cleanly flush all remaining messages in the memory queue to the sinks before terminating.
* **Extensible (SOLID):** Strictly follows the Open/Closed Principle for adding new log destinations.

### Out of Scope
* Log rotation (e.g., rolling files every 100MB), remote network log aggregation (like pushing to ELK/Splunk natively), and log formatting templates.

## 2. Tech Fundamentals & Concurrency Patterns
1. **Producer-Consumer Architecture:** Application threads are "Producers" pushing fast to an in-memory queue. A dedicated background thread is the "Consumer," waking up only when there is work to do.
2. **Strategy Pattern (Dependency Injection):** Sinks are injected via an `ILogSink` interface. The Logger engine does not know *how* to write to a file; it only knows how to execute `sink->write()`.
3. **Lock Minimization:** The background worker thread locks the queue *only* to pop a message. It then unlocks the mutex *before* writing to the slow I/O sinks, ensuring producers are never blocked by consumers.
4. **Condition Variables (`std::condition_variable`):** Used to put the background worker to sleep at the OS level when the queue is empty, preventing CPU burn (busy-waiting).

## 3. Architectural Data Flow

```mermaid
graph TD
    A[App Thread 1] -->|push msg| Q[(In-Memory Queue)]
    B[App Thread 2] -->|push msg| Q
    C[App Thread 3] -->|push msg| Q
    
    Q -->|pop msg \n cv.wait()| W[Background Worker Thread]
    
    W -->|write| S1[Console Sink]
    W -->|write| S2[File Sink]
    W -->|write| S3[Network Sink]
```

## 4. Cpp Code
### Enums & Data Models (DTOs)
```cpp
#include <iostream>
#include <string>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <memory>
#include <chrono>

using namespace std;

namespace DocuSignCore {

    // --- Enums ---
    enum class LogLevel { DEBUG, INFO, WARN, ERROR, FATAL };

    string levelToString(LogLevel level) {
        switch(level) {
            case LogLevel::DEBUG: return "DEBUG";
            case LogLevel::INFO:  return "INFO";
            case LogLevel::WARN:  return "WARN";
            case LogLevel::ERROR: return "ERROR";
            case LogLevel::FATAL: return "FATAL";
            default: return "UNKNOWN";
        }
    }

    // --- DTO ---
    struct LogMessage {
        LogLevel level;
        string message;
        long long timestamp; 
        thread::id thread_id;
    };
```
### Service Interfaces (The Strategy Pattern)
```cpp
// --- Interfaces ---
    class ILogSink {
    public:
        virtual void write(const LogMessage& msg) = 0;
        virtual ~ILogSink() = default;
    };

    // --- Concrete Sinks ---
    class ConsoleSink : public ILogSink {
    public:
        void write(const LogMessage& msg) override {
            cout << "[" << msg.timestamp << "] " 
                 << "[" << levelToString(msg.level) << "] "
                 << "[Thread: " << msg.thread_id << "] " 
                 << msg.message << "\n";
        }
    };

    class FileSink : public ILogSink {
        string filepath;
    public:
        explicit FileSink(const string& path) : filepath(path) {}
        void write(const LogMessage& msg) override {
            // Simulated file writing
            // std::ofstream file(filepath, std::ios::app);
            // file << msg.message << "\n";
        }
    };
```

### The Async Logger Engine (Producer-Consumer)
```cpp
class AsyncLogger {
    private:
        LogLevel global_level;
        vector<shared_ptr<ILogSink>> sinks;
        
        // Concurrency components
        queue<LogMessage> log_queue;
        mutex q_mtx;
        condition_variable cv;
        
        bool is_running;
        thread worker_thread;

        // The Background Consumer
        void processLogs() {
            while (true) {
                LogMessage current_msg;
                {
                    unique_lock<mutex> lock(q_mtx);
                    
                    // Sleep until queue has items OR we are shutting down
                    cv.wait(lock, [this]() { return !log_queue.empty() || !is_running; });

                    // Graceful Shutdown constraint: Drain queue before exiting
                    if (!is_running && log_queue.empty()) {
                        break;
                    }

                    current_msg = log_queue.front();
                    log_queue.pop();
                }

                // SDE-3 Detail: Execute slow I/O OUTSIDE the mutex lock
                for (auto& sink : sinks) {
                    sink->write(current_msg);
                }
            }
        }

    public:
        AsyncLogger(LogLevel level = LogLevel::INFO) : global_level(level), is_running(true) {
            worker_thread = thread(&AsyncLogger::processLogs, this);
        }

        // Graceful Shutdown (Zero Data Loss)
        ~AsyncLogger() {
            {
                lock_guard<mutex> lock(q_mtx);
                is_running = false;
            }
            cv.notify_one(); // Wake the worker to finish the queue
            if (worker_thread.joinable()) {
                worker_thread.join(); // Block main thread ONLY during shutdown until queue is empty
            }
        }

        void addSink(shared_ptr<ILogSink> sink) {
            lock_guard<mutex> lock(q_mtx); 
            sinks.push_back(sink);
        }

        // The Producer API (Ultra-low latency for application threads)
        void log(LogLevel level, const string& message) {
            if (level < global_level) return;

            LogMessage msg = {
                level, 
                message, 
                chrono::system_clock::to_time_t(chrono::system_clock::now()),
                this_thread::get_id()
            };

            {
                lock_guard<mutex> lock(q_mtx);
                log_queue.push(msg);
            }
            cv.notify_one(); // Wake the background worker
        }
        
        // Convenience wrappers
        void info(const string& msg) { log(LogLevel::INFO, msg); }
        void error(const string& msg) { log(LogLevel::ERROR, msg); }
    };

} // namespace DocuSignCore
```

### Execution Driver & Multithreading Test
```// ==========================================
// Execution Driver
// ==========================================
int main() {
    using namespace DocuSignCore;

    // Initialize with DEBUG level
    AsyncLogger logger(LogLevel::DEBUG);
    
    // Dependency Injection
    logger.addSink(make_shared<ConsoleSink>());
    logger.addSink(make_shared<FileSink>("/var/log/app.log"));

    // Simulate multi-threaded application
    auto app_task = [&logger](int id) {
        logger.info("Thread " + to_string(id) + " starting work.");
        this_thread::sleep_for(chrono::milliseconds(10)); // simulate work
        logger.error("Thread " + to_string(id) + " encountered an event.");
    };

    cout << "Starting application threads...\n";

    thread t1(app_task, 1);
    thread t2(app_task, 2);
    thread t3(app_task, 3);

    t1.join();
    t2.join();
    t3.join();

    cout << "Application threads finished. Logger will flush and safely shut down upon exiting scope.\n";

    // Logger goes out of scope here. The destructor guarantees all logs are flushed safely.
    return 0;
}
```
