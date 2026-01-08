# Resources

- [Software optimization resources](https://www.agner.org/optimize/)




## 第一阶段：夯实C++核心与性能基础（1-2个月）
### 核心目标
掌握C++中直接影响性能的核心特性（如内存管理、容器、现代C++特性），避免因语言使用不当导致的性能损耗，这是高性能编程的“地基”。

### 推荐资料
1. **核心书籍**
   - 《Effective C++》《More Effective C++》《Effective Modern C++》（Scott Meyers）：C++性能避坑的“圣经”，尤其是《Effective Modern C++》重点讲解C++11/14/17特性（移动语义、智能指针、constexpr等）的性能考量，比如“优先用emplace而非push_back”“避免不必要的拷贝”。
   - 《C++ Primer》（第5版）：若C++基础薄弱，先吃透这本，重点关注「内存管理」「标准容器实现」「函数对象」章节（比如std::vector vs std::list的性能差异根源）。
2. **补充文档**
   - [cppreference.com](https://en.cppreference.com/)：查性能相关特性的权威参考（如容器的时间复杂度、move语义的适用场景）；
   - [Abseil C++ 指南](https://abseil.io/docs/cpp/)：Google的C++最佳实践，包含大量“写出高性能、易维护代码”的编码建议。

---

## 第二阶段：吃透高性能计算的底层原理（2-3个月）
### 核心目标
理解“高性能”的本质：代码如何与CPU、内存、缓存、编译器交互。脱离硬件和编译原理的优化都是“空中楼阁”。

### 推荐资料
1. **核心书籍/文档**
   - 《Computer Systems: A Programmer's Perspective》（CSAPP，深入理解计算机系统）：必学！讲解CPU架构、缓存层次、内存访问、指令级并行、编译优化，比如“缓存行对齐”“分支预测”对性能的影响，是高性能编程的底层逻辑。
   - 《What Every Programmer Should Know About Memory》（Ulrich Drepper）：免费经典文档（可在线搜索），深入剖析内存架构与性能的关系（比如伪共享、内存带宽瓶颈），高性能计算绕不开的核心内容。
   - 《Optimized C++: Proven Techniques for Heightened Performance》：从编译器、硬件角度讲解C++代码的优化思路，比如如何让代码更易被编译器优化。
2. **实践方式**
   - 用`gcc/clang`编译代码时，通过`-S`参数生成汇编，对比`-O0/-O2/-O3`优化等级下的汇编差异，理解编译器的优化行为；
   - 做CSAPP的实验（如缓存性能测试、分支预测实验），亲手验证底层原理。

---

## 第三阶段：C++高性能编程技巧与范式（2-3个月）
### 核心目标
掌握C++特有的高性能编程技巧，写出“编译器友好、硬件友好”的代码，避开性能陷阱。

### 推荐资料
1. **核心书籍**
   - 《C++ High Performance》（第二版）：专门聚焦现代C++高性能编程，覆盖内存优化、SIMD指令、并发编程、性能测试，实践性极强（比如如何设计高效的内存池、如何利用SIMD加速数值计算）。
   - 《Performance Analysis and Tuning on Modern CPUs》：讲解现代CPU架构（如多核、超线程）对C++代码的影响，以及针对性的调优技巧。
2. **优质文章/演讲**
   - Abseil Performance Tip of the Week系列（你之前关注的[POW#9](https://abseil.io/fast/9)是其中一篇）：[全系列链接](https://abseil.io/fast)，Google工程师总结的实战优化经验，比如“避免过时的优化”“优先清晰的代码而非手写汇编”。
   - Chandler Carruth的CppCon演讲（如《Tuning C++: Benchmarks, and CPUs, and Compilers! Oh My!》）：YouTube可看，讲解如何科学地做C++性能调优，而非“凭感觉优化”。
3. **实践方式**
   - 对比不同写法的性能：比如`std::array` vs `std::vector`、手动内存池 vs `std::allocator`、循环展开 vs 编译器自动优化，用基准测试验证效果。

---

## 第四阶段：并行与分布式计算（3-4个月）
### 核心目标
高性能计算的核心是“并行”——充分利用多核CPU、GPU、分布式集群，这是突破单线程性能瓶颈的关键。

### 推荐资料
1. **核心书籍**
   - 《C++ Concurrency in Action》（Anthony Williams）：C++多线程编程的“圣经”，讲解`std::thread`、原子操作、内存模型、锁优化（如无锁编程），是并行编程的基础。
   - 《Modern C++ Concurrency in Practice》：聚焦C++17/20的并发特性（如`std::jthread`、协程`coroutine`），更贴合现代开发。
   - 《CUDA C++ Programming Guide》（NVIDIA官方）：若关注异构计算（GPU加速），这是必备，讲解CUDA C++的高性能编程技巧。
2. **核心库/工具**
   - Intel TBB：C++并行编程的经典库，封装了并行算法、任务调度，无需手动管理线程；
   - OpenMP：共享内存并行编程标准，几行指令就能实现循环并行化（适合多核CPU）；
   - MPI：分布式内存并行编程标准，适合集群高性能计算；
   - ISPC：Intel的SIMD编程工具，简化向量化指令的使用。
3. **实践方式**
   - 用TBB实现并行排序/并行遍历；
   - 用OpenMP加速数值计算循环（如矩阵乘法）；
   - 编写简单的CUDA程序（如向量加法），体验GPU高性能计算。

---

## 第五阶段：领域实战与性能调优（持续）
### 核心目标
掌握性能分析工具，能精准定位瓶颈并调优，将理论落地到实际场景（如数值计算、后端服务、游戏引擎）。

### 推荐资料
1. **性能分析工具**
   - perf（Linux）：系统级性能分析工具，分析CPU使用率、缓存命中、指令执行耗时；
   - Valgrind（Cachegrind/Callgrind）：分析缓存未命中、函数调用开销、内存泄漏；
   - Google Benchmark：C++基准测试库，量化代码性能（比如对比优化前后的吞吐量/延迟）；
   - Intel VTune：专业性能分析工具（免费社区版），精准定位瓶颈。
2. **领域案例**
   - 数值计算：Eigen库文档（高性能线性代数库），学习其内存布局、向量化设计；
   - 后端服务：brpc（百度开源RPC框架）、folly（Facebook高性能库）的源码与设计文档；
3. **实践方式**
   选一个场景（如矩阵乘法、日志处理流水线），按“朴素实现→性能分析→针对性优化（缓存对齐/SIMD/并行）→基准验证”的流程迭代，比如：
   - 优化前：朴素的嵌套循环矩阵乘法；
   - 优化1：缓存分块（解决缓存未命中）；
   - 优化2：用Eigen/SIMD加速；
   - 优化3：用OpenMP并行化。

---

## 第六阶段：跟进前沿技术（长期）
### 核心目标
保持对C++高性能计算前沿的关注，避免使用“过时的优化”（如你之前看的Abseil POW#9所提）。

### 推荐资料
1. **会议/演讲**
   - CppCon：每年的C++顶级会议，有大量高性能编程的演讲（YouTube可看）；
   - ACCU：聚焦C/C++实践，包含高性能专题。
2. **标准/工作组**
   - C++20/23新特性：如`std::execution`（并行算法）、协程、模块化对高性能的影响；
   - SG14（C++高性能工作组）：[文档链接](https://isocpp.org/community/std-proposals/sg14)，跟进高性能C++的标准提案。
3. **社区/博客**
   - Chandler Carruth（Google工程师）、Andrei Alexandrescu（元编程与高性能）的演讲/博客；
   - 知乎/掘金的高质量C++高性能专栏（筛选大厂工程师的分享）。

---

### 总结

1. **底层原理是核心**：高性能编程的本质是“让代码适配硬件和编译器”，先吃透CSAPP、内存模型、CPU架构，再谈技巧，避免“想当然的优化”；
2. **实践+工具是关键**：每一个优化思路都要用Google Benchmark、perf等工具验证效果，精准定位瓶颈而非盲目优化；
3. **循序渐进+长期跟进**：从C++基础→硬件原理→并行编程→实战调优逐步深入，同时跟进C++新标准和社区最佳实践，避免优化“过时”。