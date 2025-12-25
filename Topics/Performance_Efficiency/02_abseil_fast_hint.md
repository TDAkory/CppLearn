# Fast hints

> Jeffrey Dean & Sanjay Ghemawat, Performance Hints, 2025, https://abseil.io/fast/hints.html

## BaseLine

**完整的“过早优化”观点**：Knuth并非否定所有提前优化，而是强调**97%的场景下可忽略小效率问题，避免因过早优化引入bug和维护难题**；但**关键的3%场景中，绝不能放弃优化机会**，这份文档正是聚焦这一核心场景。

**Knuth对“小幅度优化”的立场**：很多人认为12%的性能提升微不足道，这是软件工程师对“抠细节优化导致代码不可维护”的过度反应；而在成熟工程领域，12%的易实现提升绝非边际收益，**软件工程也应秉持这一视角**——非一次性任务的高质量程序开发中，不应拒绝这类不影响代码质量的效率提升。

“先简单写代码，后续靠性能分析优化”，这种主流观点存在四大致命缺陷，导致实际中难以有效优化：

1. **性能损耗分散化**：大型系统开发中完全忽视性能，会出现**无明显热点的扁平性能剖面**，性能问题遍布各处，无法确定优化切入点。
2. **库开发的场景限制**：若开发的是供他人使用的库，使用者往往难以理解库的内部细节，也无法与开发团队协商优化，最终只能承受性能问题。
3. **系统上线后的修改成本高**：系统投入高频使用后，大幅修改代码以优化性能的难度会急剧增加。
4. **误判性能问题的解决成本**：容易忽略可轻松解决的性能问题，最终被迫采用高成本方案（如服务过度复制、资源超配）来应对负载问题。

在编写代码时，**若更快的实现方案不会显著影响代码的可读性和复杂度，应优先选择该方案**——这是平衡性能与代码质量的核心原则。

```shell
# rough costs for some basic low-level operations for estimation
L1 cache reference                             0.5 ns
L2 cache reference                             3 ns
Branch mispredict                              5 ns
Mutex lock/unlock (uncontended)               15 ns
Main memory reference                         50 ns
Compress 1K bytes with Snappy              1,000 ns
Read 4KB from SSD                         20,000 ns
Round trip within same datacenter         50,000 ns
Read 1MB sequentially from memory         64,000 ns
Read 1MB over 100 Gbps network           100,000 ns
Read 1MB from SSD                      1,000,000 ns
Disk seek                              5,000,000 ns
Read 1MB sequentially from disk       10,000,000 ns
Send packet CA->Netherlands->CA      150,000,000 ns
```

**性能分析（Profiling）的工具推荐、实用技巧**，以及的内容，明确高效的性能分析方法和针对性的优化方向。

1. **首选通用工具**：`pprof`，优势是能提供**高层级的性能信息**，且在本地开发和生产环境中都易于使用，是性能分析的入门首选。
2. **进阶深度工具**：`perf`，适合需要获取**更详细的性能洞察**（如硬件层面的性能指标）的场景。
3. **专项场景工具**：
   - 锁竞争分析：部分互斥锁（mutex）的实现自带锁竞争分析支持，可定位锁导致的CPU利用率虚低问题。
   - 机器学习场景：使用专门的ML性能分析器，适配机器学习任务的性能分析需求。

为了让性能分析和优化更高效、准确，需遵循以下实操技巧：

1. **编译配置优化**：构建生产环境的二进制文件时，要带上**适当的调试信息和优化标志**——既保留性能，又能在分析时定位到具体代码行。
2. **编写微基准测试（Microbenchmark）**：
   - 作用：缩短性能优化的周转时间，验证优化效果，防止后续性能回归。
   - 注意：微基准测试可能无法代表全系统性能（存在局限性），且有C++、Go、Java等语言的成熟库支持。
3. **利用性能计数器**：使用基准测试库输出**硬件性能计数器读数**，既能提升性能数据的精度，又能更深入地理解程序的运行行为。

**面对扁平CPU性能剖面（无明显性能热点）时的解决策略**：当CPU性能剖面呈现“扁平”状态（性能损耗分散，无明显热点），且低难度优化已完成时，可采取以下针对性策略：

1. **重视小优化的累积效应**：实现20个独立的1%优化是完全可行的，其累积效果会非常显著（这类优化依赖稳定、高质量的微基准测试支撑）。
2. **优化调用栈顶层的循环**：通过CPU剖面的**火焰图**找到调用栈顶层的循环，重构循环或其调用的代码（例如：将“增量构建复杂数据结构”改为“一次性传入全部输入构建”，减少原代码中逐元素的冗余检查）。
3. **转向结构性/算法级优化**：跳出微优化的局限，从调用栈高层寻找**结构性改进**，借助算法优化技巧（而非仅调整代码细节）解决性能问题。
4. **替换过度通用的代码**：用定制化、更低层级的实现替代过于通用的代码（例如：用简单的前缀匹配代替耗时的正则表达式匹配）。
5. **减少内存分配**：
   - 步骤：获取内存分配剖面，优先优化分配数量最多的源头。
   - 效果：①减少分配器（及GC语言的垃圾回收）的耗时；②降低缓存未命中（如使用tcmalloc的长期运行程序中，频繁分配会导致缓存行分散）。
6. **收集硬件性能计数器剖面**：借助`perf`等工具收集基于硬件性能计数器的剖面，定位到**缓存未命中率高的函数**，针对性优化。

## API considerations

### API设计的通用核心原则

1. **封装内优化，避免破坏公共接口**：性能优化优先在代码封装边界内完成（如类/模块内部），不改动对外接口；模块设计遵循“深而窄”（窄接口承载多核心功能），更容易实现这种无侵入式优化。
2. **谨慎新增API功能**：广泛使用的API新增功能会限制后续实现灵活性，还会给不需要该功能的用户增加无意义成本（如C++标准库容器的“迭代器稳定性”承诺，导致额外内存分配，多数用户并不需要）。
3. **权衡性能与易用性**：所有优化技巧需评估“性能收益”是否大于“API易用性损耗”，避免为了性能过度复杂化接口。

### 关键性能优化技巧

#### 1. 批量API（Bulk APIs）—— 摊销高频开销

减少API边界的昂贵开销（如锁、RPC、内存分配），或利用算法级优化（批量操作往往比逐次操作复杂度更低）。

- 新增`MemoryManager::LookupMany`：批量查询接口，且简化返回值（bool替代Status），贴合客户端实际需求；
- 新增`ObjectStore::DeleteRefs`：批量删除引用，单次加锁处理所有删除请求，摊销锁开销（对比逐次调用`DeleteRef`多次加锁）；

```cpp
// old impl
template <typename T>
class ObjectStore {
 public:
  ...
  absl::Status DeleteRef(Ref);
// new impl
template <typename T>
class ObjectStore {
 public:
  ...
  absl::Status DeleteRef(Ref);

  // Delete many references.  For each ref, if no other Refs point to the same
  // object, the object will be deleted.  Returns non-OK on any error.
  absl::Status DeleteRefs(absl::Span<const Ref> refs);
  ...
template <typename T>
absl::Status ObjectStore<T>::DeleteRefs(absl::Span<const Ref> refs) {
  util::Status result;
  absl::MutexLock l(&mu_);
  for (auto ref : refs) {
    result.Update(DeleteRefLocked(ref));
  }
  return result;
}
```

- 算法级批量优化：用Floyd堆构造法批量初始化堆（O(N)时间，远优于逐元素添加的O(N logN)）；
- 间接批量优化：内部缓存批量处理结果（如`lexicon.cc`缓存块解码结果，供后续非批量调用复用，避免重复解码）。(Sometimes it is hard to change callers to use a new bulk API directly. In that case it might be beneficial to use a bulk API internally and cache the results for use in future non-bulk API calls)

```cpp
// old impl
void GetTokenString(int pos, std::string* out) const {
  ...
  absl::FixedArray<LexiconEntry, 32> entries(pos + 1);

  // Decode all lexicon entries up to and including pos.
  for (int i = 0; i <= pos; ++i) {
    p = util::coding::TwoValuesVarint::Decode32(p, &entries[i].remaining,
                                                &entries[i].shared);
    entries[i].remaining_str = p;
    p += entries[i].remaining;  // remaining bytes trail each entry.
  }
mutable std::vector<absl::InlinedVector<std::string, 16>> cache_;

// new impl
void GetTokenString(int pos, std::string* out) const {
  ...
  DCHECK_LT(skentry, cache_.size());
  if (!cache_[skentry].empty()) {
    *out = cache_[skentry][pos];
    return;
  }
  ...
  // Init cache.
  ...
  const char* prev = p;
  for (int i = 0; i < block_sz; ++i) {
    uint32 shared, remaining;
    p = TwoValuesVarint::Decode32(p, &remaining, &shared);
    auto& cur = cache_[skentry].emplace_back();
    gtl::STLStringResizeUninitialized(&cur, remaining + shared);

    std::memcpy(cur.data(), prev, shared);
    std::memcpy(cur.data() + shared, p, remaining);
    prev = cur.data();
    p += remaining;
  }
  *out = cache_[skentry][pos];
```

#### 2. 视图类型（View types）—— 减少拷贝，兼容多容器

避免函数参数的不必要数据拷贝，同时兼容调用方的不同容器类型。函数参数优先使用视图类型（`std::string_view`、`std::Span<T>`、`absl::FunctionRef`），仅在需要转移数据所有权时例外；调用方可灵活选择容器（如`std::vector`/`absl::InlinedVector`），无需适配API的容器类型，且无拷贝开销。

#### 3. 预分配/预计算参数 —— 避免重复计算/分配

针对高频调用的函数，允许上层调用方传入已拥有的数据源或已计算的信息，避免底层函数重复分配临时结构、重复计算。

#### 4. 线程兼容 vs 线程安全类型 —— 按需承担同步开销

通用类型默认设计为“线程兼容”（依赖外部同步），让不需要线程安全的调用方避免承担锁等同步开销；

**例外场景**：若类型的“典型使用场景”需要同步，则将同步逻辑内置到类型中。可灵活调整同步机制（如分片锁减少竞争），且不影响调用方代码。

## Algorithmic improvements

算法级优化是性能提升的核心手段，收益远大于参数调优、硬件升级等，核心逻辑是降低算法 / 数据结构的渐近复杂度（如 O (N²)→O (N)、O (logN)→O (1)），或利用算法特性简化核心逻辑

1. **优先降低渐近复杂度**：
   这是最核心的优化（如 O(N²)→O(N)、O(logN)→O(1)），收益远大于常数级调优（如循环展开、缓存优化）。
2. **数据结构匹配场景**：
   避免“过度设计”（如用 IntervalMap 解决简单合并问题），选择“刚好满足需求”的高效结构（如哈希表）。
3. **利用算法特性简化逻辑**：
   如循环检测利用拓扑排序的 rank 对比，将复杂校验转化为简单数值比较。
4. **哈希表需配高质量哈希函数**：
   哈希表的 O(1) 是“平均复杂度”，低质量哈希会导致冲突，退化为 O(N)。
5. **空间换时间（稀疏场景友好）**：
   如死锁检测用 O(|V|+|E|) 空间换 O(1) 时间，稀疏图下空间成本可控，时间收益显著。
6. **避免人工限制**：
   旧死锁检测的 2K 限制是“性能妥协”，新算法突破限制后，既提升性能又暴露潜在问题。

## Better memory representation

优化重要数据结构的内存占用与缓存占用，减少缓存未命中、降低内存总线流量，提升程序及同主机其他程序的性能。

1. **紧凑数据结构**
   - 做法：为高频访问/占应用内存比例大的数据采用紧凑表示，降低内存使用、减少缓存行访问；
   - 注意：需警惕缓存行竞争。

2. **内存布局优化**
   - 重排字段，减少不同对齐要求字段间的填充；
   - 按需使用更小的数值类型（如枚举指定`enum class OpType : uint8_t`）；
   - 高频共访字段相邻摆放，减少常见操作的缓存行访问数；
   - 热只读字段与热可变字段分离，避免写操作驱逐只读字段出缓存；
   - 冷数据移至结构体末尾/通过间接寻址存放/放入独立数组，远离热数据；
   - 位/字节级编码压缩内存占用（仅在数据封装于测试充分的模块、内存节省效果显著时使用）；
   - 注意：位/字节编码可能导致高频数据对齐不足、访问代码开销增加，需通过基准测试验证。

3. **索引替代指针**
   - 做法：64位机器中，用数组整数索引（32位及以下）替代`T*`指针，数组元素连续存储提升缓存局部性。

4. **批量存储**
   - 做法：避免单元素单独分配（如`std::map`/`std::unordered_map`），改用分块/扁平存储结构（如`std::vector`、`absl::flat_hash_{map,set}`），降低分配器开销、优化缓存表现；可将元素分固定大小块存储（字符串、向量、`absl::flat_hash_map`均采用此思路）。

5. **内联存储**
   - 做法：使用为少量元素优化的容器（如`InlinedVector`），顶层预留小空间，元素少则无需分配；
   - 注意：`sizeof(T)`较大时不适用，内联存储会导致体积过大。

6. **消除不必要的嵌套映射**
   - 做法：将嵌套映射（如`btree<a,btree<b,c>>`）替换为复合键单层映射（`btree<pair<a,b>,c>`），降低查找/插入成本；
   - 注意：首个映射键较大时，嵌套映射可能性能更优（需要性能测试来证明）。

7. **内存池（Arenas）**
   - 做法：降低内存分配成本，将独立分配项紧凑存储以减少缓存行占用，消除大部分销毁成本（对多子对象的复杂结构效果显著）；建议设置初始大小减少分配；
   - 注意：短生命周期对象放入长生命周期内存池易导致内存膨胀。

8. **数组替代映射**
   - 做法：映射键为小整数/枚举、或元素极少时，用数组/向量替代映射（如替代`flat_map`）。

```cpp
// old impl
const gtl::flat_map<int, int> payload_type_to_clock_frequency_;
// new impl
// A map (implemented as a simple array) indexed by payload_type to clock freq
// for that paylaod type (or 0)
struct PayloadTypeToClockRateMap {
  int map[128];
};
...
const PayloadTypeToClockRateMap payload_type_to_clock_frequency_;
```

9. **位向量替代集合**
   - 做法：集合域为小整数时，用位向量（如`InlinedBitVector`）替代集合（案例：Spanner用位向量替代`dense_hash_set<ZoneId>`、位矩阵替代哈希表跟踪操作数可达性）；位运算（OR/AND等）可高效实现集合操作。

```cpp
// old impl
class ZoneSet: public dense_hash_set<ZoneId> {
 public:
  ...
  bool Contains(ZoneId zone) const {
    return count(zone) > 0;
  }

// new impl
class ZoneSet {
  ...
  // Returns true iff "zone" is contained in the set
  bool ContainsZone(ZoneId zone) const {
    return zone < b_.size() && b_.get_bit(zone);
  }
  ...
 private:
  int size_;          // Number of zones inserted
  util::bitmap::InlinedBitVector<256> b_;
```

## Reduce allocations

内存分配会产生三类成本：

1. 增加分配器执行耗时；
2. 新分配对象需高开销初始化，不再使用时可能伴随高开销销毁；
3. 每次分配通常对应新缓存行，多独立分配的数据缓存占用远大于少分配的数据（垃圾回收运行时可通过连续内存放置连续分配对象缓解此问题）。

具体优化手段

1. **避免不必要的分配**
   - 收益：减少分配可使基准测试吞吐量提升21%；
   - 做法：尽可能使用静态分配的零向量，而非分配向量后填充零；对象生命周期限于作用域时优先栈分配（需注意大对象的栈帧大小）。

2. **调整/预留容器大小**
   - 做法：提前知晓向量（或其他容器）最大/预期最大尺寸时，预调整底层存储（如C++的`resize`/`reserve`）；预分配向量并填充，而非执行N次`push_back`；
   - 注意：避免逐元素`resize`/`reserve`（易导致平方级性能损耗）；元素构造开销高时，优先先`reserve`再多次`push_back`/`emplace_back`，而非初始`resize`（后者会使构造函数调用次数翻倍）。

3. **尽可能避免拷贝**
   - 做法：优先移动而非拷贝数据结构；生命周期无问题时，临时数据结构中存储指针/索引而非对象拷贝（如本地映射存储传入proto的指针、排序索引向量而非大对象向量）；接收gRPC张量时避免额外拷贝；移动大选项结构而非拷贝；使用`std::sort`替代`std::stable_sort`（避免稳定排序的内部拷贝）。

4. **复用临时对象**
   - 做法：将循环内声明的容器/对象移至循环外（编译器常因语言语义/无法确保程序等价性无法自动优化），如循环外定义protobuf变量以复用存储、重复序列化至同一`std::string`；
   - 注意：protobuf、字符串、向量等容器会增长至存储过的最大尺寸，需定期重建（如每N次使用后），以降低内存占用和重新初始化成本。

## Avoid unnecessary work

避免无需执行的工作是提升性能的核心手段之一，具体通过为常见场景设计专用路径、预计算、延迟计算、代码特化、缓存等方式实现，同时兼顾编译器优化、统计成本控制、热路径日志管控。

1. **为常见场景设计快速路径**：代码常覆盖全量场景，但部分常见场景逻辑更简单，可针对性优化且不显著影响罕见场景性能；

2. **预计算高开销信息**（仅执行一次）：在模块边界检查输入格式是否错误，而非内部重复校验。

3. **将高开销计算移出循环**

4. **延迟高开销计算**

```cpp
// Preallocate 10 nodes not 200 for query handling in Google's web server.
// A simple change that reduced web server's CPU usage by 7.5%.

//querytree.h

static const int kInitParseTreeSize = 200;   // initial size of querynode pool
static const int kInitParseTreeSize = 10;   // initial size of querynode pool
```

5. **代码特化**：性能敏感调用点无需通用库的完整功能时，编写特化代码替代通用代码；

6. **利用缓存避免重复工作**

7. **简化编译器工作**（辅助优化）：编译器因抽象层/保守假设难以优化，开发者可底层改写辅助（仅在性能分析显示问题时操作，可通过pprof查看关键代码汇编确认优化效果）；

- 热函数中避免函数调用（减少栈帧创建成本）；
- 将慢路径代码移至独立尾调用函数；
- 大量使用前将少量数据拷贝至局部变量（消除别名，提升自动向量化和寄存器分配）；
- 手动展开极热循环；

1. **降低统计收集成本**：平衡统计信息的效用与维护成本

- 移除无用统计（如停止维护SelectServer中告警和闭包数量的高开销统计）；
- 仅对部分元素（RPC请求、输入记录等）采样维护统计（如tcmalloc分配跟踪、/requestz状态页、Dapper采样）；
- 适当降低采样率（如仅对部分文档信息请求维护统计、降低采样率并加快采样决策）。

9. **避免热代码路径中的日志**：日志语句即使未实际输出也有开销（如ABSL_VLOG的加载+比较），还可能抑制编译器优化；

## Code size considerations

性能不仅包含运行时速度，代码体积过大还会引发一系列问题：编译/链接耗时增加、二进制文件臃肿、内存占用上升、指令缓存（icache）压力增大，甚至影响分支预测器等微架构结构；该问题在C++中因过度使用模板和内联尤为突出，编写多场景复用的底层库代码、或会被多类型实例化的模板代码时，需重点考量代码体积。

1. **精简高频内联代码**：广泛调用的函数若过度内联，会显著增加代码体积；这段代码能实现 **4.5倍性能提升+代码体积缩减**，核心是通过**减少冗余代码、降低运行时开销、优化指令缓存（icache）效率**实现的，具体从3个关键优化点分析：

```cpp
// old impl
struct CheckOpString {
  CheckOpString(string* str) : str_(str) { }
  ~CheckOpString() { delete str_; }
  operator bool() const { return str_ == NULL; }
  string* str_;
};
...
#define DEFINE_CHECK_OP_IMPL(name, op) \
  template <class t1, class t2> \
  inline string* Check##name##Impl(const t1& v1, const t2& v2, \
                                   const char* names) { \
    if (v1 op v2) return NULL; \
    else return MakeCheckOpString(v1, v2, names); \
  } \
  string* Check##name##Impl(int v1, int v2, const char* names);
DEFINE_CHECK_OP_IMPL(EQ, ==)
DEFINE_CHECK_OP_IMPL(NE, !=)
DEFINE_CHECK_OP_IMPL(LE, <=)
DEFINE_CHECK_OP_IMPL(LT, < )
DEFINE_CHECK_OP_IMPL(GE, >=)
DEFINE_CHECK_OP_IMPL(GT, > )
#undef DEFINE_CHECK_OP_IMPL

// new impl
struct CheckOpString {
  CheckOpString(string* str) : str_(str) { }
  // No destructor: if str_ is non-NULL, we're about to LOG(FATAL),
  // so there's no point in cleaning up str_.
  operator bool() const { return str_ == NULL; }
  string* str_;
};
...
extern string* MakeCheckOpStringIntInt(int v1, int v2, const char* names);

template<int, int>
string* MakeCheckOpString(const int& v1, const int& v2, const char* names) {
  return MakeCheckOpStringIntInt(v1, v2, names);
}
...
#define DEFINE_CHECK_OP_IMPL(name, op) \
  template <class t1, class t2> \
  inline string* Check##name##Impl(const t1& v1, const t2& v2, \
                                   const char* names) { \
    if (v1 op v2) return NULL; \
    else return MakeCheckOpString(v1, v2, names); \
  } \
  inline string* Check##name##Impl(int v1, int v2, const char* names) { \
    if (v1 op v2) return NULL; \
    else return MakeCheckOpString(v1, v2, names); \
  }
DEFINE_CHECK_OP_IMPL(EQ, ==)
DEFINE_CHECK_OP_IMPL(NE, !=)
DEFINE_CHECK_OP_IMPL(LE, <=)
DEFINE_CHECK_OP_IMPL(LT, < )
DEFINE_CHECK_OP_IMPL(GE, >=)
DEFINE_CHECK_OP_IMPL(GT, > )
#undef DEFINE_CHECK_OP_IMPL
```

- **移除不必要的析构开销**：旧代码中`CheckOpString`类有析构函数，新代码直接删除了这个析构函数，并注释说明：**当`str_`非空时，程序会执行`LOG(FATAL)`终止，无需清理`str_`**。

- **避免模板的“重复实例化冗余**”：旧代码中`MakeCheckOpString`是**全模板实现**，对于最常见的`int-int`类型比较（`CHECK_GE(int, int)`），每次调用都会**重复实例化模板**，生成完全相同的代码（每个调用点都有一份模板实例化的副本）。

- 新代码将`int-int`场景抽离为**外部共享函数**：所有`int-int`类型的`CHECK_GE`调用，都会复用同一个`MakeCheckOpStringIntInt`函数实现，不再生成多份模板实例化的冗余代码；

2. **谨慎内联**：内联虽可能提升性能，但也可能无对应收益却增加代码体积（甚至因指令缓存压力导致性能下降）；

3. **减少模板实例化**：模板代码会随模板参数组合重复实例化，导致代码冗余；

4. **减少容器操作**：map等容器操作易生成大量代码，需关注其对体积的影响；

## Parallelization and synchronization

通过充分利用多核并行能力提升计算效率，同时优化同步机制（锁、通道等）减少开销与竞争，兼顾指令级并行（SIMD）、缓存优化等手段，平衡并行收益与同步成本。

1. **利用并行化**：将高开销任务按批次拆分后并行处理（避免逐元素并行的成本），完成后合并结果；需验证系统资源状态——无空闲CPU/内存带宽饱和时，并行化可能无效甚至性能下降。

```cpp
//Parallelization improves decoding performance by 5x.

// old impl
for (int c = 0; c < clusters->size(); c++) {
  RET_CHECK_OK(DecodeBulkForCluster(...);
}

// new impl
struct SubTask {
  absl::Status result;
  absl::Notification done;
};

std::vector<SubTask> tasks(clusters->size());
for (int c = 0; c < clusters->size(); c++) {
  options_.executor->Schedule([&, c] {
    tasks[c].result = DecodeBulkForCluster(...);
    tasks[c].done.Notify();
  });
}
for (int c = 0; c < clusters->size(); c++) {
  tasks[c].done.WaitForNotification();
}
for (int c = 0; c < clusters->size(); c++) {
  RETURN_IF_ERROR(tasks[c].result);
}
```

2. **分摊锁获取成本**：避免细粒度加锁，减少热路径中的Mutex操作开销；改动不能增加锁竞争。

3. **缩短临界区**：避免在临界区内执行高开销操作（如RPC、文件访问），减少临界区触及的缓存行数；警惕临界区解锁前执行的高开销析构函数（如~MutexUnlock触发的析构），可将高开销析构对象声明在MutexLock前（需确保线程安全）。

4. **分片（Sharding）减少锁竞争**：将高竞争的Mutex保护数据结构拆分为多个分片（各分片独立加锁，需无跨分片约束）；若为map结构，可改用并发哈希表；分片选择的信息需合理——避免哈希值部分位重复使用导致分布不均，引发哈希表性能问题。

5. **SIMD指令加速**：利用现代CPU的SIMD指令一次性处理多个数据项（如absl::flat_hash_map的批量操作场景）。

6. **减少伪共享（False Sharing）**：不同线程访问的可变数据放在不同缓存行（如C++用alignas指令），将高频修改字段与其他字段隔离在不同缓存行；alignas易滥用导致对象体积大幅增加，需性能测试验证收益。

7. **降低上下文切换频率**：小任务直接内联处理，而非提交至设备线程池。

8. **缓冲通道实现流水线**：无缓冲通道会导致写方阻塞至读方就绪，仅适用于同步，不适用于提升并行性；使用缓冲通道提升并行流水线效率。

9. **考虑无锁方案**：无锁数据结构可替代传统Mutex保护结构，优先使用高层抽象而非直接操作原子变量；直接操作原子变量易出错，需谨慎。

## Protocol Buffer advice

Protocol Buffer（Protobuf）虽便于数据的网络传输/持久化存储，但存在显著性能开销，且会增加二进制体积，引发指令缓存（i-cache）和数据缓存（d-cache）压力：

Protobuf性能优化建议

1. **避免不必要使用Protobuf**：非传输/持久化场景优先使用原生数据结构（如C++ struct+vector），而非强行用Protobuf承载数据；
2. **简化消息结构**：避免无意义的消息层级；
3. **字段设计优化**：
   - 高频字段使用小编号；
   - 谨慎选择数值类型（int32/sint32/fixed32/uint32及64位变体）；
   - proto2中，重复数值字段添加`[packed=true]`标注以打包存储；
   - 二进制数据/大值用`bytes`而非`string`；
4. **减少拷贝开销**：
   - 设`string_type = VIEW`避免字符串拷贝；
   - 大字段使用Cord类型降低拷贝成本；
5. **C++场景专属优化**：
   - 使用Protobuf Arenas（内存池）；
   - 复用Protobuf对象，避免频繁创建/销毁；
6. **工程层面优化**：
   - 保持`.proto`文件精简；
   - 内存中可直接存储序列化后的Protobuf（而非解析后的对象）；
   - 避免使用Protobuf map字段；
   - 仅使用包含字段子集的Protobuf消息定义（减少冗余）。

## C++-Specific advice

针对C++场景，通过替换标准库容器为更高效的专用数据结构（优化内存占用、缓存效率、分配开销），并限制高开销的`absl::Status/StatusOr`类型在热路径的使用，降低性能损耗。

| 数据结构                | 核心优势                               | 案例/收益                                     |
|-------------------------|-------------------|-------------------------------------------|
| absl::flat_hash_map/set | 性能优于`std::map`、`std::unordered_map`、`__gnu_cxx::hash_map`          | 加速LanguageFromCode；优化stats发布/取消发布、SelectServer告警跟踪（旧用`dense_hash_map`） |
| absl::btree_map/set     | 单节点存储多条目，减少子节点指针开销；条目内存连续，缓存效率更高          | 替代`std::set`实现高频使用的工作队列                                         |
| util::bitmap::InlinedBitVector | 短位图内联存储，性能优于`std::vector<bool>`                              | 替代`std::vector<bool>`，结合`FindNextBitSet`查找目标项                        |
| absl::InlinedVector     | 小数量元素内联存储（数量可配置），提升缓存效率，小元素场景避免内存分配    | 多处替代`std::vector`使用                                                   |
| gtl::vector32           | 仅支持32位大小的自定义vector，大幅节省内存                               | 简单类型替换后为Spanner节省约8TiB内存                                      |
| gtl::small_map          | 内联数组存储少量KV对，超出后自动切换为指定map类型                        | 在tflite_model中使用                                                      |
| gtl::small_ordered_set  | 固定数组存储少量元素，超出后切换为set/multiset，优化小集合性能            | 存储监听器集合，缩小缓存占用、缩短临界区长度                              |
| gtl::intrusive_list<T>  | 链表指针嵌入元素T内，相比`std::list<T*>`减少1个缓存行+间接寻址开销       | 跟踪每个索引行更新的未完成请求                                           |

## Further reading

In no particular order, a list of performance related books and articles that the authors have found helpful:

- [Optimizing software in C++](https://www.agner.org/optimize/optimizing_cpp.pdf) by Agner Fog. Describes many useful low-level techniques for improving performance.
- [Understanding Software Dynamics](https://www.oreilly.com/library/view/understanding-software-dynamics/9780137589692/) by Richard L. Sites. Covers expert methods and advanced tools for diagnosing and fixing performance problems.
- [Performance tips of the week](https://abseil.io/fast/) - a collection of useful tips.
- [Performance Matters](https://travisdowns.github.io/) - a collection of articles about performance.
- [Daniel Lemire’s blog](https://lemire.me/blog/) - high performance implementations of interesting algorithms.
- [Building Software Systems at Google and Lessons Learned](https://www.youtube.com/watch?v=modXC5IWTJI) - a video that describes system performance issues encountered at Google over a decade.
- [Programming Pearls and More Programming Pearls: Confessions of a Coder]() by Jon Bentley. Essays on starting with algorithms and ending up with simple and efficient implementations.
- [Hacker’s Delight](https://en.wikipedia.org/wiki/Hacker%27s_Delight) by Henry S. Warren. Bit-level and arithmetic algorithms for solving some common problems.
- [Computer Architecture: A Quantitative Approach](https://books.google.co.jp/books/about/Computer_Architecture.html?id=cM8mDwAAQBAJ&redir_esc=y) by John L. Hennessy and David A. Patterson - Covers many aspects of computer architecture, including one that performance-minded software developers should be aware of like caches, branch predictors, TLBs, etc.
