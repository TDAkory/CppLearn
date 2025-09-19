# Additional Concurrency Support in C++20

> [C++对并发的支持](https://en.cppreference.com/w/cpp/atomic.html)

- [Additional Concurrency Support in C++20](#additional-concurrency-support-in-c20)
  - [`std::jthread`](#stdjthread)
  - [协作退出](#协作退出)
  - [std::atomic\_ref](#stdatomic_ref)
  - [std::atomic\_wait](#stdatomic_wait)
  - [Semaphores](#semaphores)
  - [Latch and Barrier](#latch-and-barrier)
    - [Latch](#latch)
    - [Barrier](#barrier)
  - [新增的几个同步原语的对比](#新增的几个同步原语的对比)

根据 cppreference 的描述看，C++20新增的支持并发编程的语言工具包括如下：

* [jthread(C++20)](https://en.cppreference.com/w/cpp/thread/jthread.html) std::thread with support for auto-joining and cancellation
* Cooperative cancellation (since C++20)
* [atomic_ref(C++20)](https://en.cppreference.com/w/cpp/atomic/atomic_ref.html) provides atomic operations on non-atomic objects
* new Atomic operations [atomic_wait](https://en.cppreference.com/w/cpp/atomic/atomic_wait.html)、[atomic_nofiry_one](https://en.cppreference.com/w/cpp/atomic/atomic_notify_one.html)、[atomic_notify_all](https://en.cppreference.com/w/cpp/atomic/atomic_notify_all.html)、etc
* Semaphores (since C++20) [counting_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore.html)、[binary_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore.html)
* Latches and Barriers (since C++20) [latch](https://en.cppreference.com/w/cpp/thread/latch.html)、[barrier](https://en.cppreference.com/w/cpp/thread/barrier.html)

下面我们通过一些基本的例子来学习如何使用这些新的工具

## `std::jthread`

`std::jthread` 是 C++20 引入的线程类，继承自 `std::thread`，与 std::thread 有着相同的基本行为。

核心特性是自动管理线程生命周期（析构时会自动 `join`，避免资源泄漏），并原生支持停止令牌（stop token）机制，可协作式请求线程停止。

与 `std::thread` 相比，它解决了忘记 `join` 导致的未定义行为问题，同时通过 `std::stop_source`、`std::stop_token` 和 `std::stop_callback` 提供了安全的线程停止协作方式。

`std::jthread` 在逻辑上持有一个 `std::stop_source` 类型的内部私有成员，该成员维护一个共享的停止状态。`std::jthread` 构造函数接受一个以 `std::stop_token` 作为第一个参数的函数，`jthread` 会从其内部的 `std::stop_source` 中传递这个参数。这使得该函数可以在执行过程中检查是否已请求停止，并在请求停止时返回。

`std::jthread` 对象也可能处于不代表任何线程的状态（在默认构造、移动、分离或连接之后），一个执行线程也可能不与任何 `jthread` 对象相关联（在分离之后）。

没有两个 `std::jthread` 对象可以代表同一个执行线程；`std::jthread` 不可复制构造或复制赋值，但它可移动构造和移动赋值。

预期相关的概念如下：

* [std::jthread::get_stop_source](https://en.cppreference.com/w/cpp/thread/jthread/get_stop_source.html)
* [std::jthread::get_stop_token](https://en.cppreference.com/w/cpp/thread/jthread/get_stop_token.html)
* [std::jthread::request_stop](https://en.cppreference.com/w/cpp/thread/jthread/request_stop.html)
* [std::stop_callback](https://en.cppreference.com/w/cpp/thread/stop_callback.html)

**示例一**：普通用法，不需要像 `std::thread` 一样，关心对象在主线程退出时 `join`。[godbolt](https://godbolt.org/z/bz9dcdxYo)

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <string>

void print_hello() {
    std::cout << "Hello from jthread! (No args)\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
}

void print_message(const std::string& msg, int count) {
    for (int i = 0; i < count; ++i) {
        std::cout << "Message: " << msg << " (Count: " << i + 1 << ")\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
}

int main() {
    std::cout << "Main thread start\n";

    std::jthread t1(print_hello);

    std::jthread t2(print_message, "CppReference", 3);

    std::cout << "Main thread end\n";

    return 0;
}
```

**示例二**：协作式停止，通过 `stop_token` 检查停止请求，`request_stop()` 触发停止，实现线程安全退出。[godbolt](https://godbolt.org/z/7qcjqv91M)

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <stop_token>

void periodic_task(std::stop_token stoken, const std::string& task_name) {
    std::cout << "Task '" << task_name << "' started (press Enter to stop)\n";
    
    while (!stoken.stop_requested()) {
        std::cout << "Task '" << task_name << "' running...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
    
    std::cout << "Task '" << task_name << "' stopped gracefully\n";
}

int main() {
    std::jthread worker(periodic_task, "Heartbeat");

    std::cout << "Main thread: Before stop\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    std::cout << "Main thread: Request stop\n";
    worker.request_stop();
    std::cout << "Main thread end\n";
    return 0;
}

// Main thread: Before stop
// Task 'Heartbeat' started (press Enter to stop)
// Task 'Heartbeat' running...
// Task 'Heartbeat' running...
// Task 'Heartbeat' running...
// Task 'Heartbeat' running...
// Task 'Heartbeat' running...
// Main thread: Request stop
// Main thread end
// Task 'Heartbeat' stopped gracefully
```

**示例三**：`stop_source`可以被多个 jthread 共享。[godbolt](https://godbolt.org/z/bsaovbMz9)

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <stop_token>
#include <mutex>
#include <vector>

std::mutex cout_mutex;

void shared_task(std::stop_token stoken, int thread_id) {
    {
        std::lock_guard<std::mutex> lock(cout_mutex);
        std::cout << "Thread " << thread_id << " started\n";
    }

    while (!stoken.stop_requested()) {
        {
            std::lock_guard<std::mutex> lock(cout_mutex);
            std::cout << "Thread " << thread_id << " working\n";
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(800));
    }

    {
        std::lock_guard<std::mutex> lock(cout_mutex);
        std::cout << "Thread " << thread_id << " stopped\n";
    }
}

int main() {
    const int thread_count = 3;  
    std::vector<std::jthread> threads;
    std::stop_source global_stop_source;  

    for (int i = 0; i < thread_count; ++i) {
        threads.emplace_back(shared_task, global_stop_source.get_token(), i);
    }

    std::cout << "Main thread: Waiting 2 seconds to stop all threads...\n";
    std::this_thread::sleep_for(std::chrono::seconds(2));

    std::cout << "Main thread: Stopping all threads...\n";
    global_stop_source.request_stop();

    std::cout << "Main thread end\n";
    return 0;
}

// Thread 0 started
// Thread 0 working
// Main thread: Waiting 2 seconds to stop all threads...
// Thread 1 started
// Thread 1 working
// Thread 2 started
// Thread 2 working
// Thread 0 working
// Thread 1 working
// Thread 2 working
// Thread 0 working
// Thread 2 working
// Thread 1 working
// Main thread: Stopping all threads...
// Main thread end
// Thread 0 stopped
// Thread 1 stopped
// Thread 2 stopped
```

**示例四**：通过 std::stop_callback 在 std::jthread 退出时自动清理资源。[godbolt](https://godbolt.org/z/nda18fseW)

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <stop_token>
#include <string>

struct Resource {
    std::string name;
    Resource(const std::string& n) : name(n) {
        std::cout << "Resource '" << name << "' created\n";
    }
    ~Resource() {
        std::cout << "Resource '" << name << "' destroyed (cleanup done)\n";
    }
};

void long_running_task(std::stop_token stoken) {
    Resource task_resource("TaskBuffer");

    std::stop_callback callback(stoken, [&]() {
        std::cout << "Stop callback triggered: Preparing to stop...\n";
    });

    std::cout << "Long-running task started\n";
    int count = 0;
    while (!stoken.stop_requested() && count < 10) {
        std::cout << "Task iteration " << ++count << "\n";
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    std::cout << "Long-running task finished\n";
}

int main() {
    std::jthread task_thread(long_running_task);

    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Main thread: Requesting task stop...\n";
    task_thread.request_stop();

    std::cout << "Main thread end\n";
    return 0;
}

// Resource 'TaskBuffer' created
// Long-running task started
// Task iteration 1
// Task iteration 2
// Main thread: Requesting task stop...
// Stop callback triggered: Preparing to stop...
// Main thread end
// Long-running task finished
// Resource 'TaskBuffer' destroyed (cleanup done)
```

## 协作退出

上面提到的 `stop_source`、`stop_token` 和 `stop_callback` 组件可用于异步请求操作及时停止执行，这种请求被称为停止请求（stop request）。

这些组件规定了对**停止状态（stop state）** 进行共享访问的语义：

* 任何引用同一停止状态的 `stop_source`、`stop_token` 或 `stop_callback` 对象，分别称为**关联的**停止源、停止令牌或停止回调。
* C++26 起引入了 `stoppable-source`、`stoppable_token` 和 `stoppable-callback-for` 概念，分别规定了这些组件所需的语法和模型语义。

这些组件主要设计用于以下场景：

1. **协作式取消执行**：如 `std::jthread` 中的线程停止机制，允许安全地请求线程终止而不强制中断。
2. **中断等待函数**：可用于中断 `std::condition_variable_any` 的等待函数，实现更灵活的线程唤醒机制。
3. **异步操作的停止完成**：C++26 起支持对 `execution::connect` 创建的异步操作执行停止完成逻辑。
4. **自定义执行管理实现**：为各种并发场景提供标准化的停止协作机制。

实际上，它们甚至不需要用于"停止"操作，还可作为**线程安全的一次性函数调用触发器**使用。

从上一节的示例也可以看出，这些组件形成一个协作系统：

* `stop_source` 是停止请求的发起者，可生成 `stop_token` 并触发停止请求
* `stop_token` 是停止状态的观察者，用于检查是否有停止请求
* `stop_callback` 注册在 `stop_token` 上，当停止请求发生时自动执行

| 类别 | 名称（版本） | 描述 |
|------|------|------|
| **Stop token types** | `stop_token`(C++20) | 用于查询是否已发出 `std::jthread` 取消请求的接口（类） |
|  | `never_stop_token`(C++26) | 提供一种永远不可能也不会被请求停止的停止令牌接口（类） |
|  | `inplace_stop_token`(C++26) | 引用其关联的 `std::inplace_stop_source` 对象的停止状态的停止令牌（类） |
| **Stop source types** | `stop_source`(C++20) | 表示请求停止一个或多个 `std::jthread` 的类（类） |
|  | `inplace_stop_source`(C++26) | 作为停止状态唯一所有者的可停止源（类） |
| **Stop callback types** | `stop_callback`(C++20) | 用于在 `std::jthread` 取消时注册回调的接口（类模板） |
|  | `inplace_stop_callback`(C++26) | 用于 `std::inplace_stop_token` 的停止回调（类模板） |
|  | `stop_callback_for_t`(C++26) | 获取给定停止令牌类型的回调类型（别名模板） |
| **Concepts** (since C++20) | `stoppable_token`(C++26) | 指定允许查询停止请求和停止请求是否可能的停止令牌基本接口（概念） |
|  | `unstoppable_token`(C++26) | 指定不允许停止的停止令牌（概念） |
|  | `stoppable-source`(C++26) | 指定一种类型是关联停止令牌的工厂，并且可以对其发出停止请求（仅说明性概念*） |
|  | `stoppable-callback-for`(C++26) | 指定用于向给定停止令牌类型注册回调的接口（仅说明性概念*） |

> 注：以上内容定义于头文件 `<stop_token>` 中

## [std::atomic_ref](https://en.cppreference.com/w/cpp/atomic/atomic_ref.html)

C++11 引入的 `std::atomic` 提供了强大的原子操作支持，但它存在一个限制：必须在定义时就确定为原子类型。

C++20 带来了一个革命性的新特性 `std::atomic_ref`，允许我们对已经存在的非原子对象进行原子操作，为多线程编程提供了更大的灵活性。

`std::atomic_ref` 模板使非原子对象能够享受原子操作，无需改变其原始类型定义

在 `std::atomic_ref` 的生命周期内，被其引用的对象被视为原子对象，对该对象的读写操作遵循C++内存模型，得到线程安全的保证。

`std::atomic_ref` 引用对象的生命周期必须比 `std::atomic_ref` 自身的生命周期更长。存在 `std::atomic_ref` 引用期间，必须仅通过 `std::atomic_ref` 访问目标对象。被 `std::atomic_ref` 对象引用的任何对象的子对象，都不得被其他 `std::atomic_ref` 对象同时引用。

通过 `std::atomic_ref` 对对象应用的原子操作，与通过引用同一对象的任何其他 `std::atomic_ref` 应用的原子操作之间具有原子性。

与引用类似，`std::atomic_ref` 的常量性是浅层的——可以通过 `const` 限定的 `std::atomic_ref` 对象修改被引用的值。

> std::atomic_ref is CopyConstructible.

一个概念性的例子如下：

```cpp
#include <atomic>

int regular_int = 42;
std::atomic_ref<int> atomic_view(regular_int);

atomic_view.store(100);
int value = atomic_view.load();
```

一个计数器的例子如下：

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

int main() {
    int counter = 0;
    
    auto increment = [&counter]() {
        std::atomic_ref<int> atomic_counter(counter);
        for (int i = 0; i < 1000; ++i) {
            atomic_counter.fetch_add(1, std::memory_order_relaxed);
        }
    };
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(increment);
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    std::cout << counter << std::endl;  // 10000
    return 0;
}
```

`atomic_ref` 的构造中隐含着变量的内促对齐要求

```cpp
template< class T >
struct atomic_ref {
    //  If obj is not aligned to required_alignment, the behavior is undefined.
    explicit atomic_ref(T& obj) noexcept;
    
    atomic_ref(const atomic_ref& other) noexcept;
};

static constexpr std::size_t required_alignment = /*implementation-defined*/;   // (since C++20)

// 确保正确的内存对齐
struct alignas(std::atomic_ref<int>::required_alignment) AlignedData {
    int value;
};

AlignedData data;
std::atomic_ref<int> atomic_ref(data.value);  // 安全
```

一个性能对比的简单例子：[godbolt](https://godbolt.org/z/x9da5xz57)

```cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>

void benchmark_atomic() {
    std::atomic<int> counter(0);
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&counter]() {
            for (int j = 0; j < 1000000; ++j) {
                counter.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "std::atomic 耗时: " << duration.count() << "ms, 结果: " << counter << std::endl;
}

void benchmark_atomic_ref() {
    int counter = 0;
    auto start = std::chrono::high_resolution_clock::now();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&counter]() {
            std::atomic_ref<int> atomic_counter(counter);
            for (int j = 0; j < 1000000; ++j) {
                atomic_counter.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }
    
    for (auto& t : threads) t.join();
    
    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "std::atomic_ref 耗时: " << duration.count() << "ms, 结果: " << counter << std::endl;
}


int main() {
    benchmark_atomic();
    benchmark_atomic_ref();
}

// std::atomic 耗时: 26ms, 结果: 4000000
// std::atomic_ref 耗时: 24ms, 结果: 4000000
```

## [std::atomic_wait](https://en.cppreference.com/w/cpp/atomic/atomic_wait.html)

```cpp
// 1)
template< class T >
void atomic_wait(const std::atomic<T>* object,
                 typename std::atomic<T>::value_type old );
// 2)
template< class T >
void atomic_wait(const volatile std::atomic<T>* object,
                 typename std::atomic<T>::value_type old );
// 3)
template< class T >
void atomic_wait_explicit(const std::atomic<T>* object,
                          typename std::atomic<T>::value_type old,
                          std::memory_order order );
// 4)
template< class T >
void atomic_wait_explicit(const volatile std::atomic<T>* object,
                          typename std::atomic<T>::value_type old,
                          std::memory_order order );
```

执行原子等待操作。其行为类似于重复执行以下步骤：

1. 将 `object->load()`（对于重载 (1,2)）或 `object->load(order)`（对于重载 (3,4)）的值表示形式与 `old` 的值表示形式进行比较。
   1. 如果它们**逐位相等**，则阻塞当前线程，直到 `*object` 被 `std::atomic::notify_one()` 或 `std::atomic::notify_all()` 通知，或者线程被**伪唤醒**（spuriously unblocked）。
   2. 否则（即值不相等），立即返回。

这些函数保证只有在**值确实发生变化时**才会返回，即使底层实现发生了伪唤醒。

1,2) 等价于 `object->wait(old)`。
3,4) 等价于 `object->wait(old, order)`。

**重要限制**：如果 `order` 是 `std::memory_order::release` 或 `std::memory_order::acq_rel`，则行为是**未定义的**。

以下是一个使用 `std::atomic_wait` 的简单示例：

```cpp
#include <iostream>
#include <atomic>
#include <thread>
#include <vector>
#include <chrono>

std::atomic<int> data(0);
const int target_value = 42;

void consumer() {
    int old_value = data.load();
    
    while (old_value != target_value) {
        std::atomic_wait(&data, old_value);
        old_value = data.load();
    }
    
    std::cout << "Consumer: Data is now " << data.load() << std::endl;
}

void producer() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    data.store(target_value);
    std::atomic_notify_one(&data);  // 通知一个等待的线程
    std::cout << "Producer: Data updated and notified" << std::endl;
}

int main() {
    std::thread consumer_thread(consumer);
    std::thread producer_thread(producer);
    
    consumer_thread.join();
    producer_thread.join();
    
    return 0;
}

// Producer: Data updated and notified
// Consumer: Data is now 42
```

## [Semaphores](https://en.cppreference.com/w/cpp/thread/counting_semaphore.html)

信号量是一种轻量级的同步原语，用于限制对共享资源的并发访问。在适用场景下，信号量比条件变量更高效。

```cpp
template< std::ptrdiff_t LeastMaxValue = /* implementation-defined */ >
class counting_semaphore;

using binary_semaphore = std::counting_semaphore<1>;
```

`std::counting_semaphore` 是 C++20 引入的轻量级同步原语，用于控制对共享资源的访问。与 `std::mutex` 不同，计数信号量允许多个并发访问者同时访问同一资源，最多允许 `LeastMaxValue` 个并发访问者。

`std::binary_semaphore` 是 `std::counting_semaphore` 的特化，具体实现上，可能比 `std::counting_semaphore<1>` 更有效率

`std::counting_semaphore` 计数信号量包含一个内部计数器，该计数器在构造函数中初始化：

* 调用 `acquire()` 及相关方法会减少计数器
* 调用 `release()` 会增加计数器
* 当计数器为零时，`acquire()` 会阻塞直到计数器增加
* `try_acquire()` 不会阻塞，即使计数器为零
* `try_acquire_for()` 和 `try_acquire_until()` 会阻塞直到计数器增加或超时

`std::counting_semaphore` 的特化具有以下限制：不可默认构造 (DefaultConstructible)、不可拷贝构造 (CopyConstructible)、不可移动构造 (MoveConstructible)、不可拷贝赋值 (CopyAssignable)、不可移动赋值 (MoveAssignable)

这意味着信号量对象不能复制或移动，通常应作为全局变量或通过引用传递给线程函数。

**注意**：`LeastMaxValue` 是最小最大值，而非实际最大值。因此 `max()` 可能返回比 `LeastMaxValue` 更大的数字

与 `std::mutex` 不同，计数信号量不与执行线程绑定----获取信号量和释放信号量可以在不同的线程上进行。所有对计数信号量的操作都可以并发执行，且与特定执行线程无关，但析构操作不能并发执行（不过可以在不同线程上进行）。

信号量也常用于信号/通知语义，而不是互斥，通过将信号量初始化为 0，从而阻塞尝试调用 `acquire()` 的接收者，直到通知者通过调用 `release(n)` 发出"信号"。在这方面，信号量可以被视为 `std::condition_variable` 的替代品，通常具有更好的性能。

下面是几个使用示例：

可以使用 `binary_semaphore` 作为 “信号触发器”在线程之间进行通信。[godbolt](https://godbolt.org/z/EvxeeG9oM)

```cpp
#include <iostream>
#include <semaphore>  // C++20 头文件
#include <thread>
#include <chrono>

// 二元信号量初始化：count=0（初始为“未触发”状态，acquire() 会阻塞）
std::binary_semaphore sem_main_to_thread{0};
std::binary_semaphore sem_thread_to_main{0};

void worker_thread() {
    std::cout << "[Worker] Waiting for main thread's signal...\n";
    // 等待主线程信号：计数器从 0→1 才会继续（否则阻塞）
    sem_main_to_thread.acquire();

    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "[Worker] Task completed! Sending signal to main...\n";

    // 向主线程发送完成信号：计数器从 0→1，唤醒主线程的 acquire()
    sem_thread_to_main.release();
}

int main() {
    // 创建工作线程（此时工作线程卡在 sem_main_to_thread.acquire()）
    std::thread worker(worker_thread);

    std::cout << "[Main] Preparing data...\n";

    std::cout << "[Main] Sending start signal to worker...\n";
    sem_main_to_thread.release();

    sem_thread_to_main.acquire();
    std::cout << "[Main] Worker task done! Joining thread...\n";

    worker.join();
    return 0;
}

// [Main] Preparing data...
// [Worker] Waiting for main thread's signal...
// [Main] Sending start signal to worker...
// [Worker] Task completed! Sending signal to main...
// [Main] Worker task done! Joining thread...
```

可以使用 `binary_semaphore` 控制对共享变量的访问，防止 “数据竞争”。[godbolt](https://godbolt.org/z/Kr9cKM4j9)

```cpp
#include <semaphore>
#include <thread>
#include <vector>

int shared_counter = 0;
std::binary_semaphore sem_mutex{1};

void increment_counter(int id, int times) {
    for (int i = 0; i < times; ++i) {
        sem_mutex.acquire();
        
        shared_counter++;
        std::cout << "[Thread " << id << "] Incremented: shared_counter = " << shared_counter << "\n";
        
        sem_mutex.release();

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

int main() {
    std::thread t1(increment_counter, 1, 5);
    std::thread t2(increment_counter, 2, 5);

    t1.join();
    t2.join();

    std::cout << "[Main] Final shared_counter = " << shared_counter << "\n";  // 预期为 10
    return 0;
}

// [Thread 1] Incremented: shared_counter = 1
// [Thread 2] Incremented: shared_counter = 2
// [Thread 1] Incremented: shared_counter = 3
// [Thread 2] Incremented: shared_counter = 4
// [Thread 1] Incremented: shared_counter = 5
// [Thread 2] Incremented: shared_counter = 6
// [Thread 1] Incremented: shared_counter = 7
// [Thread 2] Incremented: shared_counter = 8
// [Thread 1] Incremented: shared_counter = 9
// [Thread 2] Incremented: shared_counter = 10
// [Main] Final shared_counter = 10
```

更一般地，可以使用 counting_semaphore 来控制对共享资源的访问并发度。

```cpp
#include <iostream>
#include <semaphore>
#include <thread>
#include <vector>
#include <chrono>

std::counting_semaphore<2> sem_task_limit{2};

void process_task(int task_id) {
    sem_task_limit.acquire();
    
    std::cout << "[" << task_id << "] Started (concurrency: " << 2 - sem_task_limit.max() + sem_task_limit.max() << ")\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    std::cout << "[" << task_id << "] Completed\n";
    sem_task_limit.release();
}

int main() {
    std::vector<std::thread> threads;
    const int total_tasks = 5;

    for (int i = 0; i < total_tasks; ++i) {
        threads.emplace_back(process_task, i + 1);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "[Main] All tasks completed!\n";
    return 0;
}

// [1] Started (concurrency: 2)
// [2] Started (concurrency: 2)
// [1] Completed
// [2] Completed
// [4] Started (concurrency: 2)
// [5] Started (concurrency: 2)
// [5] Completed
// [3] Started (concurrency: 2)
// [4] Completed
// [3] Completed
// [Main] All tasks completed!
```

## Latch and Barrier

### Latch

`std::latch` 是一个向下计数的计数器（类型为 `std::ptrdiff_t`），用于线程同步。计数器的值在创建时初始化，线程可阻塞在 `latch` 上，直到计数器递减至 0。计数器无法重置或增加，因此 `latch` 是一种 “单次使用的屏障”（single-use barrier）。综合来看，`latch` 具有以下特点

* 单次使用：计数器到 0 后，再调用 `count_down()` 或 `wait()` 无意义（计数不会再变，`wait()` 直接返回）；
* 并发安全：除析构函数外，所有成员函数支持并发调用（无数据竞争）；
* 轻量级：相比 `std::condition_variable`，`std::latch` 无需关联 `std::mutex`，实现更高效，开销更低；
* 线程无关：计数器的递减`count_down()` 和等待 `wait()` 可以在不同线程中执行（比如线程 A 递减，线程 B 等待）。

`std::latch`的成员函数

| 成员函数          | 功能描述                                                                 | 关键说明                                                                 | 适用场景                     |
|-------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------|------------------------------|
| **构造函数**      | `explicit latch(ptrdiff_t count)`                                        | 初始化计数器为 `count`（`count` 必须 ≥0，否则抛出 `std::invalid_argument`）；无默认构造，neither copyable nor movable | 创建 `latch` 实例时设定目标计数 |
| **析构函数**      | `~latch()`                                                               | 销毁 `latch`；**注意：不能在有线程等待时析构**（会导致未定义行为）。       | 生命周期结束时自动调用       |
| **count_down**    | `void count_down(ptrdiff_t update = 1)`                                  | 计数器递减 `update`（默认减 1）；无阻塞，调用后直接返回。                 | 任务完成后“通知”计数器减少   |
| **try_wait**      | `bool try_wait() const noexcept`                                         | 检查计数器是否为 0；**不阻塞**，返回 `true` 表示已到 0，`false` 表示未到。 | 非阻塞式检查同步状态         |
| **wait**          | `void wait() const`                                                      | 阻塞当前线程，直到计数器变为 0；若已为 0，直接返回。                     | 等待所有任务完成             |
| **arrive_and_wait**| `void arrive_and_wait(ptrdiff_t update = 1)`                            | 先执行 `count_down(update)`，再执行 `wait()`；相当于“完成任务后直接等待”。 | 当前线程完成任务后，等待其他线程 |
| **max()**（静态） | `static constexpr ptrdiff_t max() noexcept`                              | 返回实现支持的最大计数器值（不同编译器可能不同，通常很大）。               | 确认计数器上限               |

如果要判断编译器是否支持 `std::latch`，可检查特性测试宏 `__cpp_lib_latch`，其值为 `201907L` 表示支持（C++20 标准）：

```cpp
#if defined(__cpp_lib_latch) && __cpp_lib_latch >= 201907L
    // 支持 std::latch
#else
    // 不支持，需降级处理（如用 condition_variable）
#endif
```

下面给出一个使用 `std::latch` 实现分阶段任务同步，涉及多轮等待的例子。[godbolt](https://godbolt.org/z/5Kben1EW5)

```cpp
#include <iostream>
#include <latch>
#include <thread>
#include <vector>

void read_data(int thread_id, std::latch& read_done) {
    std::cout << "read_data " << thread_id << " start\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "read_data " << thread_id << " finish\n";
    read_done.count_down();
}

void calc_data(int thread_id, std::latch& read_done, std::latch& calc_done) {
    std::cout << "calc_data " << thread_id << " waiting...\n";
    read_done.wait();
    std::cout << "calc_data " << thread_id << " start\n";
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "calc_data " << thread_id << " finish\n";
    calc_done.count_down();
}

void output_result(std::latch& calc_done) {
    std::cout << "output_result waiting...\n";
    calc_done.wait();
    std::cout << "output_result finish\n";
}

int main() {
    std::latch read_done(2);
    std::latch calc_done(3);

    std::vector<std::thread> read_threads;
    for (int i = 1; i <= 2; ++i) {
        read_threads.emplace_back(read_data, i, std::ref(read_done));
    }

    std::vector<std::thread> calc_threads;
    for (int i = 1; i <= 3; ++i) {
        calc_threads.emplace_back(calc_data, i, std::ref(read_done), std::ref(calc_done));
    }

    std::thread output_thread(output_result, std::ref(calc_done));

    for (auto& t : read_threads) t.join();
    for (auto& t : calc_threads) t.join();
    output_thread.join();

    return 0;
}

// read_data 2 start
// calc_data 1 waiting...
// calc_data 2 waiting...
// calc_data 3 waiting...
// output_result waiting...
// read_data 1 start
// read_data 2 finish
// read_data 1 finish
// calc_data 2 start
// calc_data 3 start
// calc_data 1 start
// calc_data 1 finish
// calc_data 3 finish
// calc_data 2 finish
// output_result finish
```

### Barrier

`std::barrier` 是 C++20 标准库中引入的一种线程协调机制，用于阻塞一组已知数量的线程，直到这组线程全部抵达屏障点。与 `std::latch` 不同，屏障是可重复使用的：当一组到达的线程被解除阻塞后，屏障可以被重新使用。此外，与 `std::latch` 不同的是，屏障在解除线程阻塞之前，会执行一个可能为空的可调用函数。

一个屏障对象的生命周期由一个或多个阶段组成。每个阶段定义了一个阶段同步点，线程在此处阻塞。线程可以调用 `arrive` 方法抵达屏障，但延迟在阶段同步点上等待。这些线程稍后可以通过调用 `wait` 方法在阶段同步点上阻塞。

`std::barrier` 阶段包括以下步骤：

1. 每次调用 arrive 或 arrive_and_drop 方法时，预期计数都会减少。
2. 当预期计数达到零时，将执行阶段完成步骤，即调用完成回调函数，并解除所有在阶段同步点上阻塞的线程的阻塞。完成步骤的结束强烈发生于所有因完成步骤而解除阻塞的调用返回之前。在预期计数达到零后，恰好会有一个线程在调用 arrive、arrive_and_drop 或 wait 方法期间执行完成步骤。不过，如果没有任何线程调用 wait，则是否执行完成步骤由具体实现定义。
3. 当完成步骤结束后，预期计数将重置为构造函数中指定的值减去自上次重置以来 arrive_and_drop 的调用次数，下一个屏障阶段随即开始。

`std::barrier` 的成员函数（除析构函数外）的并发调用不会引入数据竞争。

| 特性                | std::barrier                          | std::latch                          |
|---------------------|---------------------------------------|-------------------------------------|
| **可重用性**        | ✅ 支持（自动重置计数器，多阶段同步） | ❌ 单次使用（计数器到 0 后不可变）   |
| **完成函数**        | ✅ 支持（阶段完成前执行，可选）       | ❌ 无                                |
| **计数调整**        | ✅ 支持（arrive_and_drop 减少后续阶段计数） | ❌ 仅递减，不可调整后续计数         |
| **核心场景**        | 循环任务、多阶段流程（如“工作-清理”循环） | 单次等待（如初始化、单次任务汇总） |
| **线程动态退出**    | ✅ 支持（arrive_and_drop 允许线程提前退出） | ❌ 不支持（一旦设定计数，无法减少） |

```cpp
constexpr explicit barrier( std::ptrdiff_t expected,
                            CompletionFunction f = CompletionFunction());

barrier( const barrier& ) = delete;
```

`std::barrier` 的模板参数 `CompletionFunction` 用于指定“阶段完成时执行的函数”，需满足以下要求：

* 可移动构造（MoveConstructible）、可析构（Destructible）；
* 无参数调用，且调用不抛出异常（`std::is_nothrow_invocable_v<CompletionFunction&> == true`）；
* **默认值**：若不指定，默认是一个“空操作函数”（调用后无任何效果）。

```cpp
// 完成函数：阶段完成时打印提示（noexcept 确保不抛异常）
auto on_phase_complete = []() noexcept {
    std::cout << "=== 当前阶段所有线程已到达，进入下一阶段 ===\n";
};
// 初始化 barrier：3 个线程，完成函数为 on_phase_complete
std::barrier sync_barrier(3, on_phase_complete);
```

`arrival_token`(an unspecified object type meeting requirements of MoveConstructible, MoveAssignable and Destructible)。作用：标记线程“已到达屏障”但尚未等待的状态（通过 `arrive()` 获得）；线程调用 `arrive()` 获得 `arrival_token` 后，可稍后通过 `wait(arrival_token)` 等待阶段完成（适合“先标记到达，再处理其他逻辑，最后等待”的场景）。

```cpp
auto token = sync_barrier.arrive(); // 标记到达，计数器减 1
do_something_else();                // 处理其他逻辑（不阻塞）
sync_barrier.wait(token);           // 等待阶段完成（此时若其他线程已到，直接通过）
```

`std::barrier`的成员函数

| 成员函数          | 功能描述                                                                 | 关键细节                                                                 | 适用场景                     |
|-------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------|------------------------------|
| **构造函数**      | `explicit barrier(ptrdiff_t expected, CompletionFunction f = {});`       | 初始化“预期线程数”为 `expected`（≥0，否则抛异常），完成函数为 `f`；无默认构造。 | 创建 barrier 实例             |
| **析构函数**      | `~barrier();`                                                            | 销毁 barrier；**严禁在有线程等待或执行完成函数时析构**（未定义行为）。       | 生命周期结束时自动调用       |
| **arrive()**      | `arrival_token arrive();`                                                | 标记“线程已到达”，计数器减 1；返回 `arrival_token`，**不阻塞**。             | 线程先到达，后续再等待       |
| **wait()**        | `void wait(arrival_token&& token) const;`                                | 阻塞当前线程，直到当前阶段完成；需传入 `arrive()` 返回的 `token`。          | 配合 `arrive()` 延迟等待     |
| **arrive_and_wait()** | `void arrive_and_wait();`                                              | 等价于 `wait(arrive())`：先标记到达（计数器减 1），再阻塞等待阶段完成。       | 线程到达后立即等待（最常用） |
| **arrive_and_drop()** | `void arrive_and_drop();`                                              | 1. 标记到达（当前阶段计数器减 1）；<br>2. 减少“后续阶段的预期线程数”（初始值减 1）；<br>3. 不阻塞。 | 线程提前退出，后续阶段不再等待它 |
| **max()**（静态） | `static constexpr ptrdiff_t max() noexcept;`                              | 返回实现支持的最大“预期线程数”（通常很大，如 2^31-1）。                     | 确认计数器上限               |

下面是一个使用示例：4 个线程执行 3 轮任务，其中线程 4 在第 2 轮结束后提前退出（后续阶段不再参与）。[godbolt](https://godbolt.org/z/zGWjxTW3K)

```cpp
#include <barrier>
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

auto on_phase_complete = [](int& current_thread_count) noexcept {
    static int phase = 1;
    std::cout << "\n=== " << phase << " round complete(thread count: " 
              << current_thread_count << ") ===\n";
    phase++;
};

int main() {
    int initial_thread_count = 4;
    auto completion = [&]() noexcept {
        static int phase = 1;
        std::cout << "\n=== " << phase << " round complete(thread count: " 
                  << initial_thread_count << ") ===\n";
        phase++;
    };

    std::barrier sync_barrier(initial_thread_count, completion);

    auto worker_task = [&](int thread_id) {
        for (int phase = 1; phase <= 3; ++phase) {
            if (thread_id == 4 && phase == 3) {
                std::cout << thread_id << ": exit early.\n";
                sync_barrier.arrive_and_drop();
                initial_thread_count--;
                break; 
            }

            std::cout <<  thread_id << ": " << phase << " round......\n";
            std::this_thread::sleep_for(std::chrono::seconds(1));

            sync_barrier.arrive_and_wait();

            std::cout <<  thread_id << ": " << phase << " round done\n";
        }
    };

    std::vector<std::thread> threads;
    for (int i = 1; i <= initial_thread_count; ++i) {
        threads.emplace_back(worker_task, i);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "\nall down\n";
    return 0;
}

// 1: 1 round......
// 3: 1 round......
// 4: 1 round......
// 2: 1 round......

// === 1 round complete(thread count: 4) ===
// 1: 1 round done
// 1: 2 round......
// 4: 1 round done
// 4: 2 round......
// 2: 1 round done
// 2: 2 round......
// 3: 1 round done
// 3: 2 round......

// === 2 round complete(thread count: 4) ===
// 1: 2 round done
// 1: 3 round......
// 2: 2 round done
// 2: 3 round......
// 4: 2 round done
// 4: exit early.
// 3: 2 round done
// 3: 3 round......

// === 3 round complete(thread count: 3) ===
// 3: 3 round done
// 2: 3 round done
// 1: 3 round done

// all down
```

对 `barrier` 的使用有如下的注意点：

1. 不应在有线程等待 `barrier` 或执行完成函数时析构 `barrier`，否则会触发未定义行为（程序崩溃、死锁等）。 正确做法是，确保所有线程已退出或不再使用 `barrier` 后，再让 `barrier` 生命周期结束。
2. 标准要求 `CompletionFunction` 调用时不抛出异常（`noexcept`），若抛出，程序会调用 `std::terminate()` 终止。可以在完成函数中显式添加 `noexcept`，并避免可能抛异常的操作（如 `new` 不接 `nothrow`、未捕获的异常）。
3. 调用 `arrive_and_drop()` 后，后续阶段的“预期线程数”会永久减少（初始值减去调用次数），无法恢复。若后续阶段仍需该线程参与，不要调用此函数。
4. `arrive()` 返回的 `arrival_token` 是“Move-only”类型，不能拷贝（如 `auto token2 = token1` 错误），只能移动（如 `auto token2 = std::move(token1)`）。

## 新增的几个同步原语的对比

| 同步原语          | 核心用途                  | 计数操作                | 可重用性 | 典型场景                          |
|-------------------|---------------------------|-------------------------|----------|-----------------------------------|
| **std::latch**    | 等待一组事件完成          | 仅递减（不可增加/重置） | ❌ 单次   | 主线程等待子线程初始化、分阶段任务 |
| **std::barrier**  | 多线程同步到同一执行点    | 递减到 0 后自动重置     | ✅ 可重用 | 循环执行任务（如每轮计算后同步）  |
| **std::counting_semaphore** | 控制共享资源并发访问数    | 可递增（release）/递减（acquire） | ✅ 可重用 | 限制线程池并发数、资源池管理      |