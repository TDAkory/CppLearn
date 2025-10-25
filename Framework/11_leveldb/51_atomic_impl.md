# `leveldb`的跨平台`AtomicPointer`

> 代码位置 [port/atomic_pointer.h](https://github.com/google/leveldb/blob/v1.20/port/atomic_pointer.h)

## `AtomicPointer`实现

`AtomicPointer` 主要提供了三种不同内存顺序的指针操作：

- **`NoBarrier_Load()` 和 `NoBarrier_Store()`**：不保证内存顺序，性能最高，但仅适用于无需同步的场景（例如，该指针只在单个线程内使用，或者你使用了其他外部同步机制）。
- **`Acquire_Load()`**：通常用于**读取**一个由其他线程设置的值。它确保在此操作**之后**的所有读/写操作，都不会被重排到该操作之前。这常用于进入一个临界区（例如，看到某个标志为真后，必须读取到最新的受保护数据）。
- **`Release_Store()`**：通常用于**发布/写入**一个其他线程将要读取的值。它确保在此操作**之前**的所有读/写操作，都不会被重排到该操作之后。这常用于离开一个临界区（例如，设置数据完毕后才设置标志位为真）。

```cpp
// AtomicPointer built using platform-specific MemoryBarrier()
#if defined(LEVELDB_HAVE_MEMORY_BARRIER)
class AtomicPointer {
 private:
  void* rep_;
 public:
  AtomicPointer() { }
  explicit AtomicPointer(void* p) : rep_(p) {}
  inline void* NoBarrier_Load() const { return rep_; }
  inline void NoBarrier_Store(void* v) { rep_ = v; }
  inline void* Acquire_Load() const {
    void* result = rep_;
    MemoryBarrier();
    return result;
  }
  inline void Release_Store(void* v) {
    MemoryBarrier();
    rep_ = v;
  }
};

……
```

**内存屏障防止了编译器和CPU的指令重排**，确保内存操作的顺序性。

**Acquire-Release 语义**，这是实现线程安全的核心模式：

```cpp
// 线程A：发布数据
void* data = new Data();
pointer.Release_Store(data);  // 发布操作

// 线程B：获取数据
void* loaded = pointer.Acquire_Load();  // 获取操作
```

**保证**：如果线程B通过 `Acquire_Load()` 看到了线程A通过 `Release_Store()` 存储的值，那么线程B也能看到线程A在 `Release_Store()` **之前**的所有内存写入。

场景分析：生产者-消费者模式

```cpp
// 共享数据
struct Data {
  int value1;
  int value2;
};

AtomicPointer pointer(nullptr);
Data* shared_data = nullptr;

// 生产者线程
void Producer() {
  Data* new_data = new Data();
  new_data->value1 = 100;     // 操作1
  new_data->value2 = 200;     // 操作2
  // MemoryBarrier() 在这里确保操作1和2不会重排到存储之后
  pointer.Release_Store(new_data);  // 操作3：发布指针
}

// 消费者线程
void Consumer() {
  Data* loaded = static_cast<Data*>(pointer.Acquire_Load());  // 操作4
  if (loaded) {
    // 由于acquire-release语义，我们保证能看到value1=100和value2=200
    printf("%d, %d\n", loaded->value1, loaded->value2);
  }
}
```

## `leveldb`的跨平台支持

```cpp
// Define MemoryBarrier() if available
// Windows on x86
#if defined(OS_WIN) && defined(COMPILER_MSVC) && defined(ARCH_CPU_X86_FAMILY)
// windows.h already provides a MemoryBarrier(void) macro
// http://msdn.microsoft.com/en-us/library/ms684208(v=vs.85).aspx
#define LEVELDB_HAVE_MEMORY_BARRIER

// Mac OS
#elif defined(OS_MACOSX)
inline void MemoryBarrier() {
  OSMemoryBarrier();
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// Gcc on x86
#elif defined(ARCH_CPU_X86_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  // See http://gcc.gnu.org/ml/gcc/2003-04/msg01180.html for a discussion on
  // this idiom. Also see http://en.wikipedia.org/wiki/Memory_ordering.
  __asm__ __volatile__("" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// Sun Studio
#elif defined(ARCH_CPU_X86_FAMILY) && defined(__SUNPRO_CC)
inline void MemoryBarrier() {
  // See http://gcc.gnu.org/ml/gcc/2003-04/msg01180.html for a discussion on
  // this idiom. Also see http://en.wikipedia.org/wiki/Memory_ordering.
  asm volatile("" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// ARM Linux
#elif defined(ARCH_CPU_ARM_FAMILY) && defined(__linux__)
typedef void (*LinuxKernelMemoryBarrierFunc)(void);
// The Linux ARM kernel provides a highly optimized device-specific memory
// barrier function at a fixed memory address that is mapped in every
// user-level process.
//
// This beats using CPU-specific instructions which are, on single-core
// devices, un-necessary and very costly (e.g. ARMv7-A "dmb" takes more
// than 180ns on a Cortex-A8 like the one on a Nexus One). Benchmarking
// shows that the extra function call cost is completely negligible on
// multi-core devices.
//
inline void MemoryBarrier() {
  (*(LinuxKernelMemoryBarrierFunc)0xffff0fa0)();
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// ARM64
#elif defined(ARCH_CPU_ARM64_FAMILY)
inline void MemoryBarrier() {
  asm volatile("dmb sy" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// PPC
#elif defined(ARCH_CPU_PPC_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  // TODO for some powerpc expert: is there a cheaper suitable variant?
  // Perhaps by having separate barriers for acquire and release ops.
  asm volatile("sync" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER

// MIPS
#elif defined(ARCH_CPU_MIPS_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  __asm__ __volatile__("sync" : : : "memory");
}
#define LEVELDB_HAVE_MEMORY_BARRIER

#endif
```

代码通过条件编译为多种平台实现了 `MemoryBarrier()`，其语义因平台硬件内存模型而异。

不同架构的 `MemoryBarrier` 实现差异主要源于其**内存模型**。x86/x64是**强内存模型**（TSO, Total Store Order），自身保证了大部分情况下的内存操作顺序，通常只需要编译器屏障。而ARM、PowerPC等是**弱内存模型**，允许较多的重排，因此需要显式且更强的CPU屏障指令（如 `dmb`, `sync`）来保证顺序。

### 1. Windows (x86/x64家族 + MSVC)

```cpp
// windows.h 已提供 MemoryBarrier() 宏
```

在x86/x64这种**强内存模型**的架构上，大部分情况下会保证写操作按程序顺序对其他处理器可见（即StoreStore重排较少），并且会保证读操作不会乱序（即LoadLoad和LoadStore重排较少）。但为了确保编译器和CPU都不进行重排，仍需屏障。Windows的 `MemoryBarrier()` 宏会生成一个完整的**读写屏障**，确保屏障前的所有内存访问（读写）都对屏障后的操作可见。它通常对应 `mfence` 指令或具有类似功能的指令序列。

### 2. macOS (所有支持的苹果平台)\

```cpp
inline void MemoryBarrier() {
  OSMemoryBarrier();
}
```

`OSMemoryBarrier` 是苹果系统提供的通用内存屏障，也是一个完整的**读写屏障**。它确保屏障前的所有加载和存储操作在屏障后可见。其具体实现会根据苹果设备所使用的CPU架构（如x86、ARM）而不同，但都为开发者提供了统一的强屏障语义。

### 3. x86/x64 (GCC 或 Sun Studio)

```cpp
// GCC
__asm__ __volatile__("" : : : "memory")
// Sun Studio
asm volatile("" : : : "memory")
```

这是一种**编译器内存屏障**（Compiler Barrier 或 Optimization Barrier）。`asm volatile("" : : : "memory")` 内联汇编语句中的 `"memory"` 约束会告诉编译器：**内存内容可能已被更改**，因此编译器不会将屏障前后的读写操作进行重排优化。需要注意的是，它**并不直接生成CPU级别的内存屏障指令**。在x86/x64这种具有较强内存一致性的架构上，通常不需要额外的CPU指令来防止硬件重排（某些非常宽松的情况下除外），因此这种编译器屏障通常已经足够。这也是代码注释中提到该实现非常廉价（约1ns）的原因。

### 4. ARM Linux (32位)

```cpp
inline void MemoryBarrier() {
  (*(LinuxKernelMemoryBarrierFunc)0xffff0fa0)();
}
```

ARM架构是典型的**弱内存模型**（Weak Memory Order），允许较多的指令重排。此代码通过一个固定地址 `0xffff0fa0` 直接调用Linux内核为每个用户态进程映射的**高度优化的设备特定内存屏障函数**。根据代码注释，在单核设备上，这比使用ARMv7-A的 `dmb` 指令（需要180ns）更高效；在多核设备上，函数调用的开销也可忽略。这是一个完整的读写屏障。

### 5. ARM64

```cpp
asm volatile("dmb sy" : : : "memory");
```

`dmb sy`（Data Memory Barrier, full system）是ARMv8架构提供的**数据内存屏障指令**。`sy`（full system）表示该屏障针对整个系统共享的所有内存类型。它确保在屏障**之前**发出的所有内存访问（包括缓存维护操作）对系统中**所有其他观察者**（如其他CPU核心、DMA控制器）可见之后，屏障**之后**的内存访问才会开始执行。这同样是一个完整的读写屏障，并结合了编译器屏障。

### 6. PowerPC (GCC)

```cpp
asm volatile("sync" : : : "memory");
```

`sync`（Synchronize）指令是PowerPC架构中最强的内存排序指令，它作为一个**全屏障**（Full Barrier），确保所有先前指令的效果对后续指令和所有其他处理器可见之后，才执行后续指令。这也结合了编译器屏障。

### 7. MIPS (GCC)

```cpp
__asm__ __volatile__("sync" : : : "memory");
```

MIPS架构的 `sync` 指令同样是一个**全屏障**，它使处理器等待所有未完成的内存操作和预取操作完成，确保了内存访问的全局顺序。同样结合了编译器屏障。

## 其他实现方式

**LevelDB的选择策略**：代码优先使用基于 `MemoryBarrier` 的实现（如果平台提供了廉价屏障），其次考虑C++11 `<atomic>`，最后才是特定架构的汇编实现。

```cpp
// AtomicPointer built using platform-specific MemoryBarrier()
#if defined(LEVELDB_HAVE_MEMORY_BARRIER)
    ……
// AtomicPointer based on <cstdatomic>
#elif defined(LEVELDB_ATOMIC_PRESENT)
    ……
// Atomic pointer based on sparc memory barriers
#elif defined(__sparcv9) && defined(__GNUC__)
    ……
// Atomic pointer based on ia64 acq/rel
#elif defined(__ia64) && defined(__GNUC__)
    ……
#endif
```