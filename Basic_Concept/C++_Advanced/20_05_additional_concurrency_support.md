# Additional Concurrency Support in C++20

> [C++对并发的支持](https://en.cppreference.com/w/cpp/atomic.html)

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

std::latch 是一个向下计数的计数器（类型为 std::ptrdiff_t），用于线程同步。计数器的值在创建时初始化，线程可阻塞在 latch 上，直到计数器递减至 0。计数器无法重置或增加，因此 latch 是一种 “单次使用的屏障”（single-use barrier）。



# C++20并发编程新特性：同步原语的革命性增强

C++20标准为并发编程带来了一系列重要更新，特别是新增的同步原语填补了之前标准在多线程协作方面的空白。这些标准化组件不仅简化了并发代码的编写，还提高了程序的性能和可移植性。本文将深入解析C++20中引入的核心同步机制，包括`std::latch`、`std::barrier`和`std::semaphore`，并通过实用示例展示它们如何解决实际开发中的并发挑战。

## 并发编程的演进与C++20的定位

在C++11之前，并发编程完全依赖平台特定的API（如POSIX线程或Windows线程），代码可移植性极差。C++11引入了`std::thread`、`std::mutex`等基础组件，首次为C++提供了标准化的并发支持。C++17在此基础上增加了`std::shared_mutex`等工具，但在复杂同步场景下仍显不足。

C++20的同步原语借鉴了Boost库和工业实践中的成熟方案，针对以下痛点提供了解决方案：

- 简化多线程初始化/销毁阶段的同步逻辑
- 提供可重用的线程协作机制
- 标准化信号量实现，避免重复造轮子
- 减少手动使用条件变量带来的错误风险

这些新特性遵循"零成本抽象"原则，在提供便捷性的同时不引入额外性能开销。

## std::latch：一次性同步点

`std::latch`是一个一次性使用的同步机制，允许一个或多个线程等待其他线程完成一系列操作。它的核心思想是：线程通过减少计数器表示完成某项工作，当计数器归零时，所有等待的线程被唤醒。

### 核心特性与接口

```cpp
#include <latch>

// 构造函数：指定初始计数
std::latch lat(N);

// 减少计数，不等待
lat.count_down(n);       // 减少n（默认减少1）
bool lat.try_wait();     // 若计数为0返回true，否则false

// 减少计数并等待（原子操作）
lat.arrive_and_wait(n);  // 减少n并等待计数为0（默认n=1）

// 等待计数为0
lat.wait();              // 阻塞直到计数为0
```

`std::latch`的关键特性是**不可重置性**，一旦计数器归零，后续操作不会改变其状态，这使它非常适合一次性同步场景。

### 实用示例：并行任务初始化

在多线程应用中，主线程常常需要等待所有工作线程完成初始化后再继续执行：

```cpp
#include <latch>
#include <thread>
#include <vector>
#include <iostream>

void worker_init(std::latch& init_latch, int id) {
    // 模拟初始化工作
    std::cout << "Worker " << id << " initializing...\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(100 * id));
    
    // 初始化完成，减少计数
    init_latch.count_down();
    
    // 执行其他工作（此时主线程可能仍在等待）
    std::cout << "Worker " << id << " starting work...\n";
}

int main() {
    const int num_workers = 3;
    std::latch init_latch(num_workers);  // 计数为3
    
    std::vector<std::thread> workers;
    for (int i = 0; i < num_workers; ++i) {
        workers.emplace_back(worker_init, std::ref(init_latch), i);
    }
    
    // 等待所有工作线程完成初始化
    std::cout << "Main thread waiting for initialization...\n";
    init_latch.wait();
    std::cout << "All workers initialized. Main thread proceeding.\n";
    
    // 等待工作线程完成
    for (auto& t : workers) {
        t.join();
    }
    
    return 0;
}
```

输出将显示主线程在所有工作线程初始化完成后才继续执行，完美解决了多线程启动同步问题。

### 适用场景

- 应用启动时等待所有组件初始化
- 并行算法中等待所有分块计算完成
- 测试框架中等待所有测试用例准备就绪
- 资源释放阶段等待所有使用者退出

## std::barrier：可重用的同步屏障

`std::barrier`是一种可重用的同步机制，允许固定数量的线程在每次迭代中等待彼此到达某个点。与`latch`的一次性特性不同，`barrier`在所有线程到达后可以重置，支持循环中的多次同步。

### 核心特性与接口

```cpp
#include <barrier>

// 构造函数：指定参与线程数和完成函数（可选）
std::barrier barrier(N, completion_func);

// 到达屏障并等待
barrier.arrive_and_wait();  // 到达并等待所有线程

// 到达屏障但不等待（减少等待计数）
barrier.arrive_and_drop();  // 到达并退出屏障参与
```

`std::barrier`的独特之处在于**完成函数**（completion function），当所有线程到达屏障时，会自动调用该函数（由最后一个到达的线程执行），然后所有线程被唤醒。

### 实用示例：迭代式并行计算

在分治算法或迭代优化问题中，线程需要在每轮计算后同步结果：

```cpp
#include <barrier>
#include <thread>
#include <vector>
#include <iostream>
#include <numeric>

const int NUM_THREADS = 4;
const int NUM_ITERATIONS = 3;
std::vector<double> partial_results(NUM_THREADS, 0.0);

// 屏障完成函数：汇总部分结果
void aggregate_results() {
    static int iteration = 0;
    double total = std::accumulate(partial_results.begin(), 
                                  partial_results.end(), 0.0);
    std::cout << "Iteration " << iteration++ << " complete. Total: " << total << "\n";
}

void worker_task(std::barrier<>& barrier, int thread_id) {
    for (int i = 0; i < NUM_ITERATIONS; ++i) {
        // 模拟本轮计算
        partial_results[thread_id] = (thread_id + 1) * (i + 1);
        std::cout << "Thread " << thread_id << " completed iteration " << i << "\n";
        
        // 等待所有线程完成本轮计算
        barrier.arrive_and_wait();
    }
}

int main() {
    // 创建屏障：4个线程参与，指定完成函数
    std::barrier barrier(NUM_THREADS, aggregate_results);
    
    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(worker_task, std::ref(barrier), i);
    }
    
    for (auto& t : threads) {
        t.join();
    }
    
    return 0;
}
```

这个示例展示了`barrier`在迭代计算中的优势：每轮迭代后自动汇总结果，且屏障可重复使用，避免了手动重置同步状态的麻烦。

### 适用场景

- 迭代式并行算法（如牛顿法、分形生成）
- 流水线式处理（各阶段同步后进入下一阶段）
- 仿真系统（每帧同步所有实体状态）
- 定期数据汇总与分析

## std::counting_semaphore：资源访问控制

C++20引入的`std::counting_semaphore`（计数信号量）是一种经典的同步机制，用于控制对有限资源的并发访问。它维护一个非负整数计数器，通过`acquire()`和`release()`操作管理资源的分配与释放。

### 核心特性与接口

```cpp
#include <semaphore>

// 模板参数为最大计数（编译期常量）
std::counting_semaphore<MAX_COUNT> sem(INIT_COUNT);

// 获取资源（计数器减1，若为0则阻塞）
sem.acquire();        // 可能阻塞
bool sem.try_acquire();  // 非阻塞，失败返回false

// 超时版本
bool sem.try_acquire_for(Duration d);
bool sem.try_acquire_until(TimePoint t);

// 释放资源（计数器加1）
sem.release(n);       // 增加n（默认1）
```

C++20还提供了`std::binary_semaphore`，它是`std::counting_semaphore<1>`的别名，适用于互斥访问单个资源的场景。

### 实用示例：连接池实现

数据库连接池是信号量的典型应用场景，限制同时打开的连接数量：

```cpp
#include <semaphore>
#include <vector>
#include <queue>
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

class ConnectionPool {
private:
    static const int MAX_CONNECTIONS = 5;
    std::queue<int> connections_;  // 模拟连接池
    std::mutex mtx_;
    // 信号量控制可用连接数
    std::counting_semaphore<MAX_CONNECTIONS> sem_{MAX_CONNECTIONS};

public:
    ConnectionPool() {
        // 初始化连接池
        for (int i = 1; i <= MAX_CONNECTIONS; ++i) {
            connections_.push(i);
        }
    }

    // 获取连接
    int acquire_connection() {
        sem_.acquire();  // 等待可用连接
        std::lock_guard<std::mutex> lock(mtx_);
        int conn = connections_.front();
        connections_.pop();
        std::cout << "Thread " << std::this_thread::get_id() 
                  << " acquired connection " << conn << "\n";
        return conn;
    }

    // 释放连接
    void release_connection(int conn) {
        std::lock_guard<std::mutex> lock(mtx_);
        connections_.push(conn);
        std::cout << "Thread " << std::this_thread::get_id() 
                  << " released connection " << conn << "\n";
        sem_.release();  // 增加可用连接计数
    }
};

// 模拟数据库操作
void perform_database_operation(ConnectionPool& pool, int task_id) {
    int conn = pool.acquire_connection();
    // 模拟数据库操作
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    pool.release_connection(conn);
}

int main() {
    ConnectionPool pool;
    std::vector<std::thread> tasks;

    // 启动10个任务，但最多同时使用5个连接
    for (int i = 0; i < 10; ++i) {
        tasks.emplace_back(perform_database_operation, std::ref(pool), i);
    }

    for (auto& t : tasks) {
        t.join();
    }

    return 0;
}
```

这个示例中，信号量确保了同时使用的数据库连接数不会超过最大值，有效防止了资源耗尽。

### 适用场景

- 线程池中的任务调度
- 有限资源的并发访问控制（如数据库连接、文件句柄）
- 生产者-消费者模型中的缓冲区控制
- 限制并发请求数量，防止系统过载

## C++20其他重要并发增强

除了上述同步原语，C++20还引入了其他提升并发编程体验的特性：

### std::jthread：自动管理的线程

`std::jthread`是`std::thread`的改进版，具有自动join的特性，避免了因忘记join而导致的程序终止风险：

```cpp
#include <thread>
#include <iostream>

int main() {
    // jthread析构时会自动join
    std::jthread t([]{
        std::cout << "Thread working...\n";
    });
    // 无需手动调用t.join()
    return 0;
}
```

`std::jthread`还支持通过`std::stop_token`进行协作式中断，提供了优雅的线程取消机制。

### 协程与并发

C++20引入的协程（coroutines）为异步编程提供了语言级支持，配合`std::future`和同步原语，可以编写更简洁的异步代码：

```cpp
#include <coroutine>
#include <future>
#include <iostream>

std::future<int> async_task() {
    co_return 42;  // 协程暂停并返回结果
}

int main() {
    auto fut = async_task();
    std::cout << "Result: " << fut.get() << "\n";
    return 0;
}
```

协程特别适合I/O密集型应用，避免了传统回调地狱问题。

## 最佳实践与性能考量

1. **选择合适的同步原语**：
   - 一次性同步用`latch`
   - 循环同步用`barrier`
   - 资源控制用`semaphore`

2. **避免过度同步**：
   不必要的同步会导致性能瓶颈，尽量减小临界区范围，利用原子操作替代锁机制。

3. **注意线程数量**：
   线程数超过CPU核心数会导致上下文切换开销增加，通常建议线程数等于或略大于核心数。

4. **测试并发代码**：
   使用线程 sanitizer（如`-fsanitize=thread`）检测数据竞争，通过压力测试暴露潜在的同步问题。

5. **利用编译器优化**：
   现代编译器（如GCC 10+、Clang 11+）对C++20并发特性有良好支持，启用优化（`-O2`）可显著提升性能。

## 总结

C++20引入的同步原语标志着C++并发编程进入了新的阶段。`std::latch`、`std::barrier`和`std::counting_semaphore`提供了标准化的线程协作方案，简化了代码并提高了可移植性。这些特性与`std::jthread`、协程等新功能一起，使C++在并发编程领域更加成熟和易用。

对于开发者而言，掌握这些新特性不仅能提高代码质量和性能，还能减少与平台相关的兼容性问题。随着C++标准的不断演进，我们有理由相信C++在高性能并发编程领域将继续保持领先地位。

要深入学习这些特性，建议参考：
- [cppreference.com - C++20 同步原语](https://en.cppreference.com/w/cpp/thread)
- 《C++ High Performance》第11章及附录
- C++标准委员会关于并发的提案（如P0514、P0666）