# [伪共享（False sharing）](https://en.wikipedia.org/wiki/False_sharing)

# 伪共享：多线程性能的“隐形杀手”，从原理到解决方案全解析


在多线程编程中，我们常常关注锁竞争、线程调度等显性性能问题，却容易忽视一个“隐形杀手”——**伪共享（False Sharing）**。它源于CPU缓存的底层机制，可能让精心优化的多线程代码性能暴跌，甚至不如单线程。本文将从CPU缓存原理出发，彻底讲透伪共享的本质、危害，并给出可落地的解决方案，附带完整代码示例。


## 一、前置知识：读懂CPU缓存，才能理解伪共享
伪共享的根源是CPU缓存的“缓存行（Cache Line）”机制，在深入伪共享前，必须先搞懂这部分基础。


### 1.1 CPU缓存：为什么需要它？
CPU的运算速度远超内存读写速度（差距可达100倍以上）。为了弥补这个“速度鸿沟”，CPU芯片内集成了**多级缓存**（通常是L1、L2、L3），形成“CPU → 缓存 → 内存”的分层存储架构：
- L1缓存：每个核心独占，速度最快（纳秒级），容量最小（几十KB）；
- L2缓存：每个核心独占或共享，速度次之，容量中等（几百KB）；
- L3缓存：多核心共享，速度较慢，容量较大（几MB到几十MB）。

当CPU需要数据时，会先从L1缓存查找，找不到再查L2、L3，最后查内存。缓存的存在让CPU减少了对内存的依赖，大幅提升运算效率。


### 1.2 缓存行：CPU缓存的最小单位
CPU缓存并非按“单个变量”读取，而是按**固定大小的块**读取，这个块就是“缓存行（Cache Line）”。  
- 常见缓存行大小：**64字节**（主流CPU如Intel i7、AMD Ryzen均采用此规格，部分嵌入式CPU可能为32字节或128字节）；  
- 读取逻辑：当CPU访问一个变量时，会将该变量所在的64字节连续内存块一并加载到缓存中。例如，访问一个`int`变量（4字节）时，会同时加载它相邻的15个`int`变量（共64字节）到缓存行。

缓存行的设计是为了利用“空间局部性”——程序通常会连续访问相邻内存（如数组遍历、结构体成员访问），一次加载64字节能减少缓存缺失（Cache Miss），提升效率。


### 1.3 缓存一致性协议：伪共享的“导火索”
多核心CPU中，每个核心都有自己的L1/L2缓存。当多个核心同时访问**同一缓存行**的不同变量时，需要通过“缓存一致性协议”（如MESI协议）保证数据一致性：
- 当核心A修改缓存行中的变量X时，会将该缓存行标记为“修改态（Modified）”；
- 其他核心（如核心B）若持有该缓存行的副本，会被标记为“失效态（Invalid）”；
- 核心B后续访问该缓存行的变量Y时，会发现缓存行失效，必须从核心A或内存重新加载最新的缓存行，这个过程称为“缓存行失效（Cache Line Invalidation）”。

正是这个机制，为伪共享埋下了隐患。


## 二、伪共享：是什么？为什么会坑性能？
有了CPU缓存的基础知识，我们就能轻松理解伪共享的本质。


### 2.1 伪共享的定义
当**多个线程修改不同的变量，但这些变量恰好位于同一缓存行**时，会导致缓存行频繁失效，迫使CPU反复从内存加载缓存行，这种现象就是“伪共享”。  

“伪”的含义：线程间并未共享变量（每个线程修改自己的变量），却因缓存行机制产生了“共享”的副作用——缓存行失效带来的性能损耗。


### 2.2 伪共享的产生过程（图文解析）
以“两个线程修改数组相邻元素”为例，直观展示伪共享的危害：

#### 步骤1：初始状态
- 数组`int arr[2]`的两个元素`arr[0]`和`arr[1]`位于同一64字节缓存行；
- 核心A（运行线程1）和核心B（运行线程2）的缓存中均加载了该缓存行，缓存行状态为“共享态（Shared）”。

```
核心A缓存行（Shared）：[arr[0] (4字节) | arr[1] (4字节) | 其他填充字节 (56字节)]
核心B缓存行（Shared）：[arr[0] (4字节) | arr[1] (4字节) | 其他填充字节 (56字节)]
```

#### 步骤2：线程1修改arr[0]
- 核心A修改`arr[0]`，根据MESI协议，将自己的缓存行标记为“修改态（Modified）”；
- 核心B的缓存行被标记为“失效态（Invalid）”。

```
核心A缓存行（Modified）：[arr[0]=1 | arr[1] | 其他]
核心B缓存行（Invalid）：[arr[0] | arr[1] | 其他]  // 失效，不可用
```

#### 步骤3：线程2修改arr[1]
- 核心B访问`arr[1]`时，发现缓存行失效，必须从核心A或内存重新加载最新的缓存行；
- 加载后，核心B的缓存行变为“共享态”，修改`arr[1]`后再次标记为“修改态”，核心A的缓存行又失效。

#### 步骤4：循环往复，性能暴跌
线程1和线程2反复修改各自的变量，导致缓存行在“修改→失效→重新加载”之间频繁切换，CPU大量时间浪费在缓存同步上，而非运算本身——这就是伪共享的危害。


## 三、伪共享的危害：性能损耗有多严重？
伪共享的性能损耗并非理论空谈，在高并发场景下（如多线程计数、线程池任务处理），性能差距可能达到**数倍甚至十倍**。下面通过一个实验量化其影响。


### 3.1 实验场景：多线程累加计数器
设计两个版本的计数器：
- **普通版本**：两个线程分别累加`Counter`结构体的`a`和`b`成员（大概率在同一缓存行）；
- **无伪共享版本**：通过缓存行填充，让`a`和`b`各自独占一个缓存行。

#### 代码实现（C++）
```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

// 普通计数器（可能存在伪共享）
struct NormalCounter {
    volatile int a = 0;  // volatile防止编译器优化
    volatile int b = 0;
};

// 无伪共享计数器（缓存行填充）
// 假设缓存行大小为64字节，int占4字节，需填充64-4=60字节
struct PaddingCounter {
    volatile int a = 0;
    // 填充60字节（15个int，每个4字节）
    volatile int pad[15] = {0};  
    volatile int b = 0;
};

// 累加函数：循环count次，每次加1
template <typename Counter>
void increment(Counter& cnt, bool is_a, int count) {
    for (int i = 0; i < count; ++i) {
        if (is_a) {
            cnt.a++;
        } else {
            cnt.b++;
        }
    }
}

// 测试函数：返回运行时间（毫秒）
template <typename Counter>
long long test(int thread_num, int count) {
    Counter cnt;
    std::vector<std::thread> threads;
    auto start = std::chrono::high_resolution_clock::now();

    // 启动线程：一半线程累加a，一半累加b
    for (int i = 0; i < thread_num; ++i) {
        threads.emplace_back(increment<Counter>, std::ref(cnt), (i % 2 == 0), count);
    }

    // 等待所有线程完成
    for (auto& t : threads) {
        t.join();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    return duration.count();
}

int main() {
    const int THREAD_NUM = 4;  // 4线程
    const int COUNT = 10'000'000;  // 每个线程累加1000万次

    // 测试普通版本（伪共享）
    auto normal_time = test<NormalCounter>(THREAD_NUM, COUNT);
    // 测试无伪共享版本（缓存行填充）
    auto padding_time = test<PaddingCounter>(THREAD_NUM, COUNT);

    std::cout << "伪共享版本运行时间：" << normal_time << " ms\n";
    std::cout << "无伪共享版本运行时间：" << padding_time << " ms\n";
    std::cout << "性能提升：" << (normal_time - padding_time) * 100.0 / normal_time << "%" << std::endl;

    return 0;
}
```

#### 编译与运行
需支持C++11及以上，编译时添加`-pthread`（Linux）或`/MT`（Windows）：
```bash
g++ -std=c++11 false_sharing.cpp -o false_sharing -pthread
./false_sharing
```


#### 实验结果（Intel i7-12700H，Linux系统）
```
伪共享版本运行时间：89 ms
无伪共享版本运行时间：23 ms
性能提升：74.16%
```

**结论**：伪共享导致性能损耗超过70%！无伪共享版本因缓存行不冲突，CPU缓存命中率大幅提升，运算效率显著提高。


## 四、攻克伪共享：4种实用解决方案
针对伪共享的本质（变量位于同一缓存行），解决方案的核心思路是：**让多线程修改的变量各自独占一个缓存行**，或避免变量被多线程共享。下面介绍4种落地性强的方案。


### 4.1 方案1：缓存行填充（Cache Line Padding）
最直接的方案：在变量前后添加“无用填充字节”，让变量的总占用空间达到缓存行大小（如64字节），确保其独占一个缓存行。

#### 代码示例（通用版）
```cpp
// 通用缓存行填充模板（适配不同缓存行大小）
template <size_t CacheLineSize = 64, typename T>
struct CacheAligned {
    T data;
    // 计算填充字节数：CacheLineSize - sizeof(T)
    static constexpr size_t PadSize = (CacheLineSize > sizeof(T)) ? (CacheLineSize - sizeof(T)) : 1;
    // 填充字节（用volatile防止编译器优化掉）
    volatile char pad[PadSize] = {0};
};

// 使用示例：两个int变量各自独占64字节缓存行
CacheAligned<64, int> var1;
CacheAligned<64, int> var2;

// 多线程修改：var1和var2在不同缓存行，无伪共享
std::thread t1([&]() { for (int i = 0; i < 1e8; ++i) var1.data++; });
std::thread t2([&]() { for (int i = 0; i < 1e8; ++i) var2.data++; });
```

#### 注意事项
- 缓存行大小需适配目标CPU（可通过`sysconf(_SC_LEVEL1_DCACHE_LINESIZE)`在Linux下获取实际缓存行大小）；
- 用`volatile`修饰填充字节，防止编译器因“填充字节未被使用”而优化掉（否则伪共享问题依然存在）。


### 4.2 方案2：编译器对齐指令（标准化方案）
手动计算填充字节不够优雅，现代编译器提供了**对齐指令**，可直接指定变量的对齐方式，确保独占缓存行。主流方案有两种：

#### （1）C++11及以后：`alignas`关键字（跨平台）
`alignas(N)`表示变量按`N`字节对齐，`N`需为2的幂（如64）。
```cpp
// 按64字节对齐，确保var独占一个缓存行
alignas(64) volatile int var = 0;

// 结构体成员对齐示例
struct AlignedStruct {
    alignas(64) volatile int a = 0;  // a独占64字节缓存行
    alignas(64) volatile int b = 0;  // b独占64字节缓存行
};
```

#### （2）GCC/Clang：`__attribute__((aligned(N)))`（Linux）
GCC扩展属性，功能与`alignas`类似：
```cpp
// 按64字节对齐
volatile int var __attribute__((aligned(64))) = 0;

// 结构体对齐
struct GCCAlignedStruct {
    volatile int a __attribute__((aligned(64))) = 0;
    volatile int b __attribute__((aligned(64))) = 0;
};
```

#### 优势
- 无需手动计算填充字节，代码更简洁；
- 编译器会自动处理对齐，避免人为计算错误；
- `alignas`是C++标准，跨平台兼容性优于编译器扩展。


### 4.3 方案3：数据结构设计优化
通过调整数据布局，从根源上避免多线程访问的变量进入同一缓存行。常见思路有两种：

#### （1）数组分块（Array Chunking）
多线程遍历数组时，将数组按“缓存行大小”分块，每个线程处理一个块，避免跨块访问。
```cpp
const size_t CACHE_LINE_SIZE = 64;
const size_t INT_SIZE = sizeof(int);
// 每个块包含64/4=16个int（独占一个缓存行）
const size_t CHUNK_SIZE = CACHE_LINE_SIZE / INT_SIZE;

int arr[1024];

// 线程1处理块0：arr[0]~arr[15]（同一缓存行）
std::thread t1([&]() {
    for (int i = 0; i < CHUNK_SIZE; ++i) {
        arr[i]++;
    }
});

// 线程2处理块1：arr[16]~arr[31]（另一缓存行）
std::thread t2([&]() {
    for (int i = CHUNK_SIZE; i < 2*CHUNK_SIZE; ++i) {
        arr[i]++;
    }
});
```

#### （2）稀疏数据结构（Sparse Data）
将多线程访问的变量分散存储，而非集中在连续内存。例如，用哈希表替代数组，让变量的内存地址不连续，降低同一缓存行的概率。


### 4.4 方案4：线程本地存储（TLS）
如果变量无需跨线程共享，可将其放入“线程本地存储（Thread Local Storage, TLS）”，每个线程拥有独立的变量副本，从根源上避免共享，自然不存在伪共享。

#### 代码示例（C++11 `thread_local`）
```cpp
// 线程本地变量：每个线程有独立的counter副本
thread_local int counter = 0;

// 线程函数：修改自己的counter，无共享
void thread_func(int count) {
    for (int i = 0; i < count; ++i) {
        counter++;
    }
    // 线程内访问自己的counter
    std::cout << "Thread " << std::this_thread::get_id() << " counter: " << counter << "\n";
}

int main() {
    std::thread t1(thread_func, 1e6);
    std::thread t2(thread_func, 2e6);
    t1.join();
    t2.join();
    return 0;
}
```

#### 适用场景
- 变量仅需在单个线程内使用（如线程私有计数器、临时缓存）；
- 无需跨线程同步，性能开销极低（TLS访问速度接近普通局部变量）。


## 五、如何检测伪共享？工具与方法
伪共享的隐蔽性强，仅凭代码难以判断是否存在。下面介绍两种实用的检测方法：


### 5.1 性能分析工具：直接定位缓存问题
主流性能分析工具能监控“缓存行失效次数”“缓存命中率”等指标，若这些指标异常，大概率存在伪共享。

#### （1）Linux：`perf`（开源免费）
`perf`是Linux内核自带的性能分析工具，可监控缓存事件：
```bash
# 监控缓存行失效事件（L1D_CACHE_REFILLS:MODIFIED）
perf stat -e L1D_CACHE_REFILLS:MODIFIED ./false_sharing
```

#### （2）Intel VTune（商用，有免费社区版）
VTune是Intel推出的性能分析工具，能直观展示“缓存冲突”“伪共享”等问题，甚至定位到具体代码行。


### 5.2 对比实验法：间接验证
通过“有无伪共享”两个版本的性能对比，间接判断是否存在伪共享：
1. 实现一个基础版本（未处理伪共享）；
2. 实现一个优化版本（如添加缓存行填充）；
3. 对比两者的运行时间，若优化版本性能提升显著（如超过30%），则说明原版本存在伪共享。


## 六、注意事项：避免优化陷阱
在解决伪共享时，需避免以下3个常见陷阱：


### 6.1 不要盲目填充：缓存行大小可能不同
不同CPU的缓存行大小可能不同（如嵌入式CPU为32字节，部分服务器CPU为128字节），盲目按64字节填充可能导致：
- 填充不足：仍存在伪共享；
- 填充过度：浪费内存（如大量变量时，内存占用翻倍）。

**解决方法**：通过代码动态获取缓存行大小（如Linux下`sysconf(_SC_LEVEL1_DCACHE_LINESIZE)`），或使用编译器对齐指令（`alignas`会自动适配）。


### 6.2 编译器优化可能“破坏”填充
编译器的“死代码消除”优化可能会移除未被使用的填充字节，导致伪共享问题复发。  
**解决方法**：用`volatile`修饰填充字节，或在代码中“触碰”填充字节（如初始化时赋值），确保编译器不优化。


### 6.3 权衡性能与内存开销
缓存行填充会增加内存占用（如一个`int`变量从4字节变为64字节，内存扩大16倍）。在变量数量极少时，内存开销可忽略；但在变量数量庞大（如百万级数组）时，过度填充可能导致内存不足，反而影响性能。

**解决方法**：仅对高并发访问的核心变量进行填充，非核心变量无需处理。


## 七、总结：伪共享优化的核心思路
伪共享的本质是“CPU缓存行机制”与“多线程变量布局”不匹配导致的性能损耗。优化的核心思路可归纳为3点：
1. **隔离变量**：让多线程修改的变量各自独占缓存行（填充、对齐）；
2. **优化布局**：调整数据结构，避免敏感变量进入同一缓存行（分块、稀疏存储）；
3. **避免共享**：用TLS让变量私有化，从根源上消除共享。

在多线程编程中，伪共享虽隐蔽，但只要理解CPU缓存原理，结合工具检测和针对性优化，就能有效规避。记住：**高性能多线程代码，既要关注上层逻辑，也要理解底层硬件机制**。


```cpp
#include <iostream>
#include <thread>
#include <new>
#include <atomic>
#include <chrono>
#include <latch>
#include <vector>

using namespace std;
using namespace chrono;

#if defined(__cpp_lib_hardware_interference_size)
// default cacheline size from runtime
constexpr size_t CL_SIZE = hardware_constructive_interference_size;
#else
// most common cacheline size otherwise
constexpr size_t CL_SIZE = 64;
#endif

int main()
{
    vector<jthread> threads;
    int hc = jthread::hardware_concurrency();
    hc = hc <= CL_SIZE ? hc : CL_SIZE;
    for (int nThreads = 1; nThreads <= hc; ++nThreads)
    {
        // synchronize beginning of threads coarse on kernel level
        latch coarseSync(nThreads);
        // fine synch via atomic in userspace
        atomic_uint fineSync(nThreads);
        // as much chars as would fit into a cacheline
        struct alignas(CL_SIZE) { char shareds[CL_SIZE]; } cacheLine;
        // sum of all threads execution times
        atomic_int64_t nsSum(0);
        for (int t = 0; t != nThreads; ++t)
            threads.emplace_back(
                [&](char volatile &c)
                {
                    coarseSync.arrive_and_wait(); // synch beginning of thread execution on kernel-level
                    if (fineSync.fetch_sub(1, memory_order::relaxed) != 1) // fine-synch on user-level
                        while (fineSync.load(memory_order::relaxed));
                    auto start = high_resolution_clock::now();
                    for (size_t r = 10'000'000; r--;)
                        c = c + 1;
                    nsSum += duration_cast<nanoseconds>(high_resolution_clock::now() - start).count();
                }, ref(cacheLine.shareds[t]));
        threads.resize(0); // join all threads
        cout << nThreads << ": " << (int)(nsSum / (1.0e7 * nThreads) + 0.5) << endl;
    }
}
```