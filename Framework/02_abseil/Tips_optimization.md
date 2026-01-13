# Tips of Optimization

## Optimizing for application productivity

本文是Google发布的性能优化技巧系列第7篇，核心主题是**以“应用生产力”为核心进行优化**，而非依赖传统资源消耗或指令相关指标。

### 核心背景与核心指标

- Google服务器集群的核心目标是完成“有用工作”（如处理搜索查询、转码视频等），而非单纯降低资源消耗（如CPU占用、内存分配耗时）。
- 单纯的资源消耗指标（如CPU使用率上升）无法区分“工作负载增加”（如处理更多视频）和“效率下降”（如无用的协议转换），因此需要**应用生产力指标（APMs）** ——即每单位资源（CPU秒、内存字节秒、磁盘操作、硬件加速器时间）完成的实际工作总量（如处理的视频数量），以此衡量优化价值。

### 传统指标的局限性（需APMs补充的场景）

传统指标（如MIPS、IPC、组件相对耗时）无法准确反映以下关键优化的价值，甚至可能误导决策：

1. **核心基础设施变更优化**
   - TCMalloc的大页感知分配器（Temeraire）：虽增加了TCMalloc自身的相对耗时，但通过优化数据布局提升了整体应用性能，属于正向优化。
   - Protocol Buffer的Arenas分配：不仅加速了协议缓冲区代码（如消息析构），还优化了业务逻辑的数据局部性，使主流框架每CPU能多处理15-30%的工作。
2. **新指令集应用**
   - 新硬件指令（如`rep movsb`、AVX、FMA、BMI）虽可能降低MIPS（每秒百万指令数）或IPC（每周期指令数），但单指令能完成更多工作（如`rep movsb`替代复制循环），实际提升处理效率。例如，用`--march=haswell`编译启用新指令集时，虽MIPS下降，但查询耗时减少、QPS提升。
3. **编译器优化**
   - 内联等优化会减少动态执行的指令数，虽降低“指令总量”相关指标，但简化了代码并加速执行，提升生产力。
4. **内核优化**
   - 调整大页策略、线程调度等内核参数可能增加内核自身开销（如内存压缩工作量），但应用层面的收益远大于这些成本。

### 核心结论

- 应用生产力指标（APMs）是衡量优化价值的关键，能帮助团队识别传统指标无法覆盖的有效优化。
- 优化的核心目标是“提升单位资源的有用工作量”，而非单纯降低某一组件的耗时或提升指令相关指标，这一导向能更高效地优化Google服务器集群的整体效率。

## Optimizations past their prime

本文核心观点为：优化并非一劳永逸，过去针对旧平台（如Intel Pentium 3、AMD Opterons）的合理优化，随着硬件迭代与编译器发展，可能逐渐沦为性能负担。文章通过两个具体案例展开，并给出长期有效的优化实践建议。

### 1. Popcount指令相关优化

- 背景：2008年Intel引入`popcnt`指令（用于统计整数中置1位的数量），Google相关库提供两个实现：`CountOnes64`（运行时检测指令可用性，支持则使用，否则降级）和`CountOnes64withPopcount`（x86_64平台无条件通过内联汇编使用`popcnt`）。
- 问题：随着硬件普及，所有x86_64机器均支持`popcnt`指令，但`CountOnes64`的运行时检测开销仍存在；且内联汇编会阻止编译器规避部分处理器的假依赖bug，而使用编译器内建函数的`CountOnes64`反而能生成更优机器码，导致原本旨在选“快实现”的运行时调度，实际选择了更慢的版本。

### 2. CHECK_EQ宏优化

- 背景：2005年Google基于`CheckOpString`（`std::string*`的轻量包装）实现`CHECK`日志宏，后续添加了编译器优化提示（预测比较结果大概率为真）。
- 问题：LLVM编译时会产生冗余检查——先判断`a == b`（预测为真），再判断`str_ != nullptr`（预测为假），但`a == b`为真时`str_`必然是`nullptr`，冗余分支源于多年累积的代码复杂度；而调试版本早在2008年就已移除`CheckOpString`，优化版本后续移除该包装后，直接使用`std::string*`，消除了冗余开销。

### 最佳实践建议

1. 优先编写清晰、符合语言习惯的代码：不仅便于阅读和调试，长期来看也更易被编译器优化。
2. 谨慎使用低层级优化：对于位操作、 intrinsics 代码、内联汇编等低层级优化，先判断编译器是否能自主实现；若必须使用，按“便携性优先级”选择（hwy生成便携向量代码→intrinsics→内联汇编，内联汇编应极罕见），避免降低代码跨架构兼容性。
3. 保留参考实现：优化时不要删除被替换的“朴素”代码，可将其作为参考函数（如`REFERENCE_ComputeFoo`），方便后续单元测试（验证功能一致性）、微基准测试和必要时回滚。
4. 附带微基准测试：优化变更需搭配对应的微基准测试，确保优化效果。
5. 合理设计配置项：配置项应基于“预期结果”而非“具体行为”设计，避免固定过时的默认逻辑，确保配置能随环境变化持续保持最优。

## Fixing things with hashtable profiling

核心结论：Abseil C++哈希表（如`flat_hash_map`）内置性能分析工具，可通过关键统计指标识别哈希函数低熵、碰撞过多等问题，结合盐值注入（Using a distinct hash function by salting the hash）、更换哈希函数等手段，能显著优化哈希表操作性能（如插入、查找），文中通过多个生产环境案例验证了该方法的有效性。

- 数据收集：从生产环境服务器收集哈希表性能数据，通过`pprof`工具加载分析。
- 核心目标：定位“有问题的哈希表”，这类哈希表通常源于两个原因——哈希函数熵值低（混合效果差）、分片与表内查找复用同一哈希函数。
- 基础优化方向：优先使用`absl::Hash`（自带良好的位混合效果）；通过“盐值注入”实现分片与表内查找的哈希函数差异化。

**问题1：Stuck Bits（卡住的位）**：Stuck Bits指所有哈希值中“始终不变”的位（要么全0，要么全1），会导致哈希碰撞概率呈指数级上升，严重影响哈希表操作效率。

问题场景：哈希表数组`pending_ack_callbacks_`（256个分片）通过`Closure*`地址哈希取模分片，分片内哈希表复用同一哈希函数，导致所有分片的哈希值低位完全相同（`0xff`），碰撞频发。

解决方案：给分片哈希注入盐值，使用`std::pair(地址, 固定值)`作为哈希输入，让分片与表内查找使用不同哈希函数。

```cpp
// 优化后：盐值42实现哈希函数差异化
absl::Hash<std::pair<Closure*, int>> h;
return h(std::pair(address, 42)) % kNumPendingAckShards;
```

优化效果：该库的插入操作成本降低50%。

**问题2：高碰撞率（过多探测次数）**：哈希表查找时，若目标位置被占用需“探测”后续位置，理想探测长度为0（一次命中）。当平均探测长度>0.1（10%的键发生碰撞），或最大探测长度过大时，查找性能会从O(1)退化至O(n)，且可能触发未缓存的内存加载，进一步降低效率。

- 平均探测长度计算：`total_probe_length / (num_erases + size)`（分子为总探测次数，分母为累计插入元素数）。

问题场景：`priority_hash_map`（封装`absl::flat_hash_map`）使用自定义哈希函数，通过`xor`组合`IpFlow`的IP、端口等字段，位混合效果差，导致最大探测长度达133，平均探测长度8.4，近似线性查找。
  
解决方案：为`IpFlow`实现`AbslHashValue`接口，接入`absl::Hash`的位混合逻辑，替代原有的`xor`组合方式。
  
```cpp
// 优化后：通过H::combine实现高效位混合
template<typename H>
friend H AbslHashValue(H h, const IpFlow& flow) {
  return H::combine(std::move(h), flow.remote_ip(), flow.proto(), 
                    flow.local_port(), flow.remote_port());
}
```

**关键哈希表统计指标（Profiling核心数据）**：分析工具会记录每个哈希表的以下指标，用于定位问题：

| 指标名称 | 核心意义 |
|----------|----------|
| `capacity` | 哈希表的实际容量 |
| `size` | 当前存储的元素数量 |
| `max_probe_length` | 单个元素的最大探测长度（反映最坏碰撞情况） |
| `total_probe_length` | 所有元素的探测长度总和（计算平均探测长度的基础） |
| `stuck_bits` | 哈希值中“卡住的位”掩码（>10元素的表理想值为0） |
| `num_rehashes` | 重哈希次数（合理调用`reserve`可减少该值） |
| `max_reserve` | 调用`reserve`时传入的最大容量（判断容量是否过度预留） |
| `num_erases` | 累计删除元素数（与`size`之和为累计插入数） |
| `inline_element_size`/`key_size`/`value_size` | 元素、键、值的大小（影响内存布局与访问效率） |
| `soo_capacity` | 小对象优化（SOO）的元素容量 |
| `table_age` | 哈希表创建时长（微秒） |

单元测试中的问题检测

- 启用方式：设置环境变量`ABSL_CONTAINER_FORCE_HASHTABLEZ_SAMPLE=1`，运行单元测试时自动触发哈希表性能检测。
- 注意事项：检测阈值较高，因单元测试插入的元素数量通常较少，可能无法触发碰撞场景，仅作为额外防护手段。

核心优化原则总结

1. 优先使用`absl::Hash`，其内置高效位混合逻辑，避免自定义哈希函数的低熵问题；
2. 分片场景下，通过“盐值注入”让分片哈希与表内哈希函数差异化，避免Stuck Bits；
3. 定期通过`pprof`分析哈希表指标，重点关注`stuck_bits`（需为0）和平均探测长度（需<0.1）；
4. 合理调用`reserve`设置预期容量，减少重哈希次数和内存浪费。

> Finding issues in hashtables from a CPU profile alone is challenging since opportunities may not appear prominently in the profile. SwissMap’s built-in hashtable profiling lets us peer into a number of hashtable properties to find low-quality hash functions, collisions, or excessive rehashing.

## Beware microbenchmarks bearing gifts

1. **基准测试的核心定位与层级**
   - 核心观点：基准测试不能精准预测生产环境性能，仅用于辅助效率调试；性能测试的预测性远低于正确性测试，微基准测试的误导风险更高。
   - 层级划分（从简单到复杂、从快到慢、从低精度到高精度）：微基准测试 → 单任务负载测试 → 集群负载测试 → 生产环境，层级越高越贴近真实场景，目的是优化性能优化的迭代开发过程（缩短“OODA循环”）。

2. **微基准测试误导生产的典型案例**
   - **memcmp代码膨胀**：glibc的memcmp实现为微基准优化采用手写汇编，虽单测性能优，但代码量达~6KB，导致指令缓存失效、驱逐其他函数，宏观性能下降；类似地，微基准中“更快”的memcpy实现因代码 footprint 大，反而比“看似较慢”的小体积实现表现更差。
   - **算术运算vs预计算数组加载**：微基准中预计算掩码数组可缓存常驻，看似优化，但生产环境中数组缓存可能被驱逐（引发主存访问阻塞），且x86 Haswell后的BMI2指令集（如bzhi）比加载+掩码更高效。
   - **TCMalloc预取操作**：微基准显示该预取操作成本高（占new快速路径70%+周期），但生产中预取的下一个同尺寸对象常被复用，且能在无依赖的预取阶段处理TLB缺失，避免使用时阻塞，实际提升CPU生产力。

3. **微基准测试的常见陷阱**
   - 数据/指令缓存不具代表性：基准测试工作集小、缓存常驻，而生产环境多为内存受限、指令 footprint 庞大（谷歌代码特征）；
   - 敏感于底层细节：函数/分支对齐、栈对齐的微小变化可能导致20%性能波动；
   - 测试场景不匹配：数据类型、代码路径与生产差异大，或引入真实 workload 中不存在的代码（如冗余随机数生成）；
   - 忽略动态行为：基准测试易收敛到稳态，而生产负载存在变异性。

4. **合理利用基准测试的策略**
   - 聚焦组件单个属性：测试核心操作（如搜索场景中posting list的迭代创建/销毁、前进等基础行为）；
   - 锚定生产行为：通过生产 profiling 确定关键优化点，避免优化非核心操作；
   - 多层级验证：先通过微基准验证优化原型，再用单任务、集群负载测试逐步验证，若结果矛盾需修正微基准（可能未测准目标）；
   - 重视相对性能：即使部分操作生产占比低，其基准测试可提供相对性能参考，避免优化导致该操作性能退化（如用迭代替代前进操作的可行性分析）。

5. **总结**
   - 微基准测试虽可辅助原型验证，但难以准确反映生产性能；
   - 采用“从简单到复杂”的多层级基准测试体系，理解基准测试与生产环境的差异，是提升测试保真度、避免误导的关键，同时能反向加深对生产行为的认知。

## Ref

- [Performance Tip of the Week #7: Optimizing for application productivity](https://abseil.io/fast/7)
- [Performance Tip of the Week #9: Optimizations past their prime](https://abseil.io/fast/9)
- [Performance Tip of the Week #26: Fixing things with hashtable profiling](https://abseil.io/fast/26)
- [Performance Tip of the Week #39: Beware microbenchmarks bearing gifts](https://abseil.io/fast/39)
