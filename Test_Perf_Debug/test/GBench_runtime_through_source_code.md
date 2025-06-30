# GBench runtime through source code

- [google/benchmark](https://github.com/google/benchmark)
- [Mastering C++ with Google Benchmark](https://ashvardanian.com/posts/google-benchmark/)

Google Benchmark 是一个由 Google 开发的 C++ 基准测试库，它用于测量和报告代码的性能指标，如执行时间、速率等。Benchmark 库提供了一个简单易用的框架，允许开发者编写基准测试来评估他们的代码在不同条件下的表现。

benchmark工程的readme中提供了一个简单是示例如下：

```cpp
#include <benchmark/benchmark.h>

static void BM_StringCreation(benchmark::State& state) {
  for (auto _ : state)
    std::string empty_string;
}
// Register the function as a benchmark
BENCHMARK(BM_StringCreation);

// Define another benchmark
static void BM_StringCopy(benchmark::State& state) {
  std::string x = "hello";
  for (auto _ : state)
    std::string copy(x);
}
BENCHMARK(BM_StringCopy);

BENCHMARK_MAIN();
```

显而易见，`google/benchmark`的核心就是`BENCHMAKR`宏，我们接下来就来分析，这个宏展开，究竟做了什么。

## `BENCHMARK`做了什么

```cpp
#define BENCHMARK(...)                                               \
  BENCHMARK_PRIVATE_DECLARE(_benchmark_) =                           \
      (::benchmark::internal::RegisterBenchmarkInternal(             \
          new ::benchmark::internal::FunctionBenchmark(#__VA_ARGS__, \
                                                       __VA_ARGS__)))
```

这里有一个宏、一个函数、一个类型，我们逐个分析：

### `BENCHMARK_PRIVATE_DECLARE`

```cpp
#define BENCHMARK_PRIVATE_DECLARE(n)                                 \
  static ::benchmark::internal::Benchmark* BENCHMARK_PRIVATE_NAME(n) \
      BENCHMARK_UNUSED

#define BENCHMARK_PRIVATE_NAME(...)                                      \
  BENCHMARK_PRIVATE_CONCAT(benchmark_uniq_, BENCHMARK_PRIVATE_UNIQUE_ID, \
                           __VA_ARGS__)

// Check that __COUNTER__ is defined and that __COUNTER__ increases by 1
// every time it is expanded. X + 1 == X + 0 is used in case X is defined to be
// empty. If X is empty the expression becomes (+1 == +0).
#if defined(__COUNTER__) && (__COUNTER__ + 1 == __COUNTER__ + 0)
#define BENCHMARK_PRIVATE_UNIQUE_ID __COUNTER__
#else
#define BENCHMARK_PRIVATE_UNIQUE_ID __LINE__
#endif

#define BENCHMARK_PRIVATE_CONCAT(a, b, c) BENCHMARK_PRIVATE_CONCAT2(a, b, c)
#define BENCHMARK_PRIVATE_CONCAT2(a, b, c) a##b##c
```

可以看到，这里其实是定义了一个静态的`::benchmark::internal::Benchmark*`类型的指针，`BENCHMARK_PRIVATE_DECLARE(_benchmark_)`会被展开为：

```cpp
// 这里我们假设 __COUNTER__ 此时累加到1
static ::benchmark::internal::Benchmark* benchmark_uniq_1__benchmark_
```

这就构成了BENCHMARK展开的等号左边的类型，等号右边是一个函数调用，我们先来分析函数的参数。

### `FunctionBenchmark`

```cpp
new ::benchmark::internal::FunctionBenchmark(#__VA_ARGS__, __VA_ARGS__)
```

这个表达式得到了一个`::benchmark::internal::FunctionBenchmark`类型的指针，其构造函数的入参，则是目标函数名、目标函数指针。如下可见这个类型很简单，是`Benchmark`的派生，额外保存了函数指针。

```cpp
// The class used to hold all Benchmarks created from static function.
// (ie those created using the BENCHMARK(...) macros.
class BENCHMARK_EXPORT FunctionBenchmark : public Benchmark {
 public:
  FunctionBenchmark(const std::string& name, Function* func)
      : Benchmark(name), func_(func) {}

  void Run(State& st) BENCHMARK_OVERRIDE;

 private:
  Function* func_;
};
```

### `RegisterBenchmarkInternal`

接下来我们看注册的部分，其函数签名是`BENCHMARK_EXPORT Benchmark* RegisterBenchmarkInternal(Benchmark*);`

```cpp
Benchmark* RegisterBenchmarkInternal(Benchmark* bench) {
  std::unique_ptr<Benchmark> bench_ptr(bench);
  BenchmarkFamilies* families = BenchmarkFamilies::GetInstance();
  families->AddBenchmark(std::move(bench_ptr));
  return bench;
}
```

```cpp
// Class for managing registered benchmarks.  Note that each registered
// benchmark identifies a family of related benchmarks to run.
class BenchmarkFamilies {
  ...

  std::vector<std::unique_ptr<Benchmark>> families_;
  Mutex mutex_;
};
```

`BenchmarkFamilies`是一个全局单例，并利用`vector`来保存构建出来的`Benchmark`，每一个用户定义的被测试函数，被成为一个`family`，根据参数的不同，可能会扩展出多个实际的压测对象。

这样就结束了么？非也，在我们使用`BENCHMARK(...)`的时候，是可以给这个`Benchmark`指定参数的，比如`Arg`,`Range`,`Threads`,`Iterations`等待，这些参数又是如何保存、如何生效的呢？

## `class Benchmark`

在前面的小结其实我们已经注意到，`class BENCHMARK_EXPORT FunctionBenchmark : public Benchmark`，这里的基类`Benchmark`可以回答上面的问题：

```cpp
// ------------------------------------------------------
// Benchmark registration object.  The BENCHMARK() macro expands
// into an internal::Benchmark* object.  Various methods can
// be called on this object to change the properties of the benchmark.
// Each method returns "this" so that multiple method calls can
// chained into one expression.
class BENCHMARK_EXPORT Benchmark {

    ...

private:
  std::string name_;
  AggregationReportMode aggregation_report_mode_;
  std::vector<std::string> arg_names_;       // Args for all benchmark runs
  std::vector<std::vector<int64_t> > args_;  // Args for all benchmark runs

  TimeUnit time_unit_;
  bool use_default_time_unit_;

  int range_multiplier_;
  double min_time_;
  double min_warmup_time_;
  IterationCount iterations_;
  int repetitions_;
  bool measure_process_cpu_time_;
  bool use_real_time_;
  bool use_manual_time_;
  BigO complexity_;
  BigOFunc* complexity_lambda_;
  std::vector<Statistics> statistics_;
  std::vector<int> thread_counts_;

  typedef void (*callback_function)(const benchmark::State&);
  callback_function setup_;
  callback_function teardown_;
};
```

可以看到这个基类包含了benchmark支持的各种可配置参数，这些参数都会保存在`Bencmark`的成员变量里面，我们以`Range`,`Threads`两中参数类型为例，来分析下其存储方式

```cpp
Benchmark* Benchmark::Range(int64_t start, int64_t limit) {
  BM_CHECK(ArgsCnt() == -1 || ArgsCnt() == 1);
  std::vector<int64_t> arglist;
  AddRange(&arglist, start, limit, range_multiplier_);

  for (int64_t i : arglist) {
    args_.push_back({i});
  }
  return this;
}
```

可以看到`Range`调用了帮助函数`AddRange`，将我们的入参展开为一组实际参数，然后保存在成员变量`args_`里面

```cpp
Benchmark* Benchmark::Threads(int t) {
  BM_CHECK_GT(t, 0);
  thread_counts_.push_back(t);
  return this;
}
```

线程数则保存在成员变量`thread_counts_`里面，注意这里使用`vector`来保存每一种线程数，这会影响后续生成的实际的`benchmark`个数。

## `BENCHMARK_MAIN()`

benchmark的入口是`BENCHMARK_MAIN()`：

```cpp
// Helper macro to create a main routine in a test that runs the benchmarks
// Note the workaround for Hexagon simulator passing argc != 0, argv = NULL.
#define BENCHMARK_MAIN()                                                \
  int main(int argc, char** argv) {                                     \
    char arg0_default[] = "benchmark";                                  \
    char* args_default = arg0_default;                                  \
    if (!argv) {                                                        \
      argc = 1;                                                         \
      argv = &args_default;                                             \
    }                                                                   \
    ::benchmark::Initialize(&argc, argv);                               \
    if (::benchmark::ReportUnrecognizedArguments(argc, argv)) return 1; \
    ::benchmark::RunSpecifiedBenchmarks();                              \
    ::benchmark::Shutdown();                                            \
    return 0;                                                           \
  }                                                                     \
  int main(int, char**)
```

其中，`::benchmark::Initialize`的核心就是解析了命令行参数，并设置了日志级别

```cpp
void Initialize(int* argc, char** argv, void (*HelperPrintf)()) {
  internal::HelperPrintf = HelperPrintf;
  internal::ParseCommandLineFlags(argc, argv);
  internal::LogLevel() = FLAGS_v;
}
```

完成初始化之后，还需要判断是否存在未识别的命令行参数，会输出日志并中断执行。

如果一切正常，则会执行benchmark。可以看到，命令行参数`benchmark_filter`会作为过滤条件传入，用来匹配需要运行的目标函数，如果该参数为空，则传递下去的是`.`来进行正则匹配。

```cpp
size_t RunSpecifiedBenchmarks() {
  return RunSpecifiedBenchmarks(nullptr, nullptr, FLAGS_benchmark_filter);
}

size_t RunSpecifiedBenchmarks(BenchmarkReporter* display_reporter,
                              BenchmarkReporter* file_reporter,
                              std::string spec) {
  if (spec.empty() || spec == "all")
    spec = ".";  // Regexp that matches all benchmarks

  // Setup the reporters
  ...

  std::vector<internal::BenchmarkInstance> benchmarks;
  if (!FindBenchmarksInternal(spec, &benchmarks, &Err)) {
    Out.flush();
    Err.flush();
    return 0;
  }

  if (benchmarks.empty()) {
    Err << "Failed to match any benchmarks against regex: " << spec << "\n";
    Out.flush();
    Err.flush();
    return 0;
  }

  if (FLAGS_benchmark_list_tests) {
    for (auto const& benchmark : benchmarks)
      Out << benchmark.name().str() << "\n";
  } else {
    internal::RunBenchmarks(benchmarks, display_reporter, file_reporter);
  }

  Out.flush();
  Err.flush();
  return benchmarks.size();
}
```

排除一些分支逻辑和输出逻辑，这里的两个核心函数是：`FindBenchmarksInternal`, `RunBenchmarks`。

```cpp
// FIXME: This function is a hack so that benchmark.cc can access
// `BenchmarkFamilies`
bool FindBenchmarksInternal(const std::string& re,
                            std::vector<BenchmarkInstance>* benchmarks,
                            std::ostream* Err) {
  return BenchmarkFamilies::GetInstance()->FindBenchmarks(re, benchmarks, Err);
}

bool BenchmarkFamilies::FindBenchmarks(
    std::string spec, std::vector<BenchmarkInstance>* benchmarks,
    std::ostream* ErrStream) {
  ...

  // Special list of thread counts to use when none are specified
  const std::vector<int> one_thread = {1};

  int next_family_index = 0;

  MutexLock l(mutex_);
  for (std::unique_ptr<Benchmark>& family : families_) {
    int family_index = next_family_index;
    int per_family_instance_index = 0;

    // Family was deleted or benchmark doesn't match
    if (!family) continue;

    // 如果用户没有传入任何参数，这里会构造空参数传入
    if (family->ArgsCnt() == -1) {              
      family->Args({});
    }
    // 压测的线程数：若用户未配置则为1，否则为用户配置
    const std::vector<int>* thread_counts =
        (family->thread_counts_.empty()
             ? &one_thread
             : &static_cast<const std::vector<int>&>(family->thread_counts_));
    // 一个被测函数扩展出来的、真实的测试对象数量：是 参数个数 与 线程数 的乘积（参数向量和线程向量的叉乘）
    const size_t family_size = family->args_.size() * thread_counts->size();
    // The benchmark will be run at least 'family_size' different inputs.
    // If 'family_size' is very large warn the user.
    if (family_size > kMaxFamilySize) {
      Err << "The number of inputs is very large. " << family->name_
          << " will be repeated at least " << family_size << " times.\n";
    }
    // reserve in the special case the regex ".", since we know the final
    // family size.  this doesn't take into account any disabled benchmarks
    // so worst case we reserve more than we need.
    
    // 如果命令行的过滤条件是空，即通配，则扩展benchmarks，为insert做准备
    if (spec == ".") benchmarks->reserve(benchmarks->size() + family_size);

    for (auto const& args : family->args_) {
      for (int num_threads : *thread_counts) {
        // 在这里进行了参数和线程的组合，构造了一个测试对象`BenchmarkInstance`
        BenchmarkInstance instance(family.get(), family_index,
                                   per_family_instance_index, args,
                                   num_threads);

        const auto full_name = instance.name().str();
        // 此时会根据构造的测试对象的名字，来进行过滤。通过的才会加入到benchmarks里面
        if (full_name.rfind(kDisabledPrefix, 0) != 0 &&
            ((re.Match(full_name) && !is_negative_filter) ||
             (!re.Match(full_name) && is_negative_filter))) {
          benchmarks->push_back(std::move(instance));

          ++per_family_instance_index;

          // Only bump the next family index once we've estabilished that
          // at least one instance of this family will be run.
          if (next_family_index == family_index) ++next_family_index;
        }
      }
    }
  }
  return true;
}
```

非常直观的，`FindBenchmarksInternal`通过遍历`benchmark_families`(由用户通过`Benchmark(...)`注入)，然后遍历每个`benchmark_family`的 **参数**、**线程** 组合，通过正则匹配来过滤本次需要运行的测试实例`BenchmarkInstance`，并返回这些过滤结果，交给`RunBenchmarks`来运行。

```cpp
void RunBenchmarks(const std::vector<BenchmarkInstance>& benchmarks,
                   BenchmarkReporter* display_reporter,
                   BenchmarkReporter* file_reporter) {
  // Note the file_reporter can be null.
  ...

  // Determine the width of the name field using a minimum width of 10.
  // 一些格式对齐和美化的工作
  ...

  // Print header here
  ...

  // Keep track of running times of all instances of each benchmark family.
  std::map<int /*family_index*/, BenchmarkReporter::PerFamilyRunReports>
      per_family_reports;

  if (display_reporter->ReportContext(context) &&
      (!file_reporter || file_reporter->ReportContext(context))) {
    ...

    size_t num_repetitions_total = 0;

    // This perfcounters object needs to be created before the runners vector
    // below so it outlasts their lifetime.
    PerfCountersMeasurement perfcounters(
        StrSplit(FLAGS_benchmark_perf_counters, ','));

    // Vector of benchmarks to run
    std::vector<internal::BenchmarkRunner> runners;
    runners.reserve(benchmarks.size());

    // Count the number of benchmarks with threads to warn the user in case
    // performance counters are used.
    int benchmarks_with_threads = 0;

    // Loop through all benchmarks
    for (const BenchmarkInstance& benchmark : benchmarks) {
      BenchmarkReporter::PerFamilyRunReports* reports_for_family = nullptr;
      if (benchmark.complexity() != oNone)
        reports_for_family = &per_family_reports[benchmark.family_index()];
      benchmarks_with_threads += (benchmark.threads() > 1);
      // 遍历benchmark并构造runner
      runners.emplace_back(benchmark, &perfcounters, reports_for_family);
      ...
    }
    assert(runners.size() == benchmarks.size() && "Unexpected runner count.");

    // The use of performance counters with threads would be unintuitive for
    // the average user so we need to warn them about this case
    if ((benchmarks_with_threads > 0) && (perfcounters.num_counters() > 0)) {
      GetErrorLogInstance()
          << "***WARNING*** There are " << benchmarks_with_threads
          << " benchmarks with threads and " << perfcounters.num_counters()
          << " performance counters were requested. Beware counters will "
             "reflect the combined usage across all "
             "threads.\n";
    }

    // 根据每个 runner 的重复次数，把 runner_index 重复构造，并塞入到 repetition_indices 中
    std::vector<size_t> repetition_indices;
    repetition_indices.reserve(num_repetitions_total);
    for (size_t runner_index = 0, num_runners = runners.size();
         runner_index != num_runners; ++runner_index) {
      const internal::BenchmarkRunner& runner = runners[runner_index];
      std::fill_n(std::back_inserter(repetition_indices),
                  runner.GetNumRepeats(), runner_index);
    }
    assert(repetition_indices.size() == num_repetitions_total &&
           "Unexpected number of repetition indexes.");

    // 如果打开了乱序运行的配置，则对 repetition_indices 执行一次 shuffle
    if (FLAGS_benchmark_enable_random_interleaving) {
      std::random_device rd;
      std::mt19937 g(rd());
      std::shuffle(repetition_indices.begin(), repetition_indices.end(), g);
    }

    for (size_t repetition_index : repetition_indices) {
      internal::BenchmarkRunner& runner = runners[repetition_index];
      runner.DoOneRepetition();
      if (runner.HasRepeatsRemaining()) continue;
      // FIXME: report each repetition separately, not all of them in bulk.

      // some report job
      ...
  }
  ...
}
```

`RunBenchmarks`的核心逻辑就是通过遍历前面生成的`BenchmarkInstance`，来生成对应的`BenchmarkRunner`对象，并逐个执行这些Runner。

```cpp
void BenchmarkRunner::DoOneRepetition() {
  ...

  // We *may* be gradually increasing the length (iteration count)
  // of the benchmark until we decide the results are significant.
  // And once we do, we report those last results and exit.
  // Please do note that the if there are repetitions, the iteration count
  // is *only* calculated for the *first* repetition, and other repetitions
  // simply use that precomputed iteration count.
  for (;;) {
    b.Setup();
    i = DoNIterations();
    b.Teardown();

    // Do we consider the results to be significant?
    // If we are doing repetitions, and the first repetition was already done,
    // it has calculated the correct iteration time, so we have run that very
    // iteration count just now. No need to calculate anything. Just report.
    // Else, the normal rules apply.
    const bool results_are_significant = !is_the_first_repetition ||
                                         has_explicit_iteration_count ||
                                         ShouldReportIterationResults(i);

    if (results_are_significant) break;  // Good, let's report them!

    // Nope, bad iteration. Let's re-estimate the hopefully-sufficient
    // iteration count, and run the benchmark again...

    iters = PredictNumItersNeeded(i);
    assert(iters > i.iters &&
           "if we did more iterations than we want to do the next time, "
           "then we should have accepted the current iteration run.");
  }

  // Produce memory measurements if requested.
  ...

  // Ok, now actually report.
  ...
}
```

`DoNIterations()`负责根据当前`BenchmarkInstance`的线程配置，在线程池中执行一次测试。

当用户没有通过参数设置一个`benchmark`执行多少个循环的时候，具体执行的循环个数是有程序自己决定的，通过`ShouldReportIterationResults`来判断何时终止当前Runner的循环。

以上就是Benchmark的执行流程了。

## CPU开销是如何统计的

`BenchmarkRunner::DoNIterations()` 根据当前 `BenchmarkInstance` 的线程配置，在线程池中执行一次测试。该函数会创建 `ThreadManager` 来管理线程的执行，并启动多个线程并行执行测试。

```cpp
BenchmarkRunner::IterationResults BenchmarkRunner::DoNIterations() {
  

  std::unique_ptr<internal::ThreadManager> manager;
  manager.reset(new internal::ThreadManager(b.threads()));

  // Run all but one thread in separate threads
  for (std::size_t ti = 0; ti < pool.size(); ++ti) {
    pool[ti] = std::thread(&RunInThread, &b, iters, static_cast<int>(ti + 1),
                           manager.get(), perf_counters_measurement_ptr,
                           /*profiler_manager=*/nullptr);
  }
  // And run one thread here directly.
  // (If we were asked to run just one thread, we don't create new threads.)
  // Yes, we need to do this here *after* we start the separate threads.
  RunInThread(&b, iters, 0, manager.get(), perf_counters_measurement_ptr,
              /*profiler_manager=*/nullptr);

  // The main thread has finished. Now let's wait for the other threads.
  manager->WaitForAllThreads();
  ...
  return i;
}

// Execute one thread of benchmark b for the specified number of iterations.
// Adds the stats collected for the thread into manager->results.
void RunInThread(const BenchmarkInstance* b, IterationCount iters,
                 int thread_id, ThreadManager* manager,
                 PerfCountersMeasurement* perf_counters_measurement,
                 ProfilerManager* profiler_manager_) {
  internal::ThreadTimer timer(
      b->measure_process_cpu_time()
          ? internal::ThreadTimer::CreateProcessCpuTime()
          : internal::ThreadTimer::Create());

  State st = b->Run(iters, thread_id, &timer, manager,
                    perf_counters_measurement, profiler_manager_);
  BM_CHECK(st.skipped() || st.iterations() >= st.max_iterations)
      << "Benchmark returned before State::KeepRunning() returned false!";
  {
    MutexLock l(manager->GetBenchmarkMutex());
    internal::ThreadManager::Result& results = manager->results;
    results.iterations += st.iterations();
    results.cpu_time_used += timer.cpu_time_used();
    results.real_time_used += timer.real_time_used();
    results.manual_time_used += timer.manual_time_used();
    results.complexity_n += st.complexity_length_n();
    internal::Increment(&results.counters, st.counters);
  }
  manager->NotifyThreadComplete();
}
```

`ThreadTimer` 是统计 CPU 开销的关键类。在 `RunInThread` 函数中，会根据 `BenchmarkInstance` 的 `measure_process_cpu_time()` 方法的返回值来决定创建的 `ThreadTimer` 的属性，它们的区别仅体现在获取的是线程耗时还是进程耗时：

```cpp
  static ThreadTimer Create() {
    return ThreadTimer(/*measure_process_cpu_time_=*/false);
  }
  static ThreadTimer CreateProcessCpuTime() {
    return ThreadTimer(/*measure_process_cpu_time_=*/true);
  }

  ……

  double ReadCpuTimerOfChoice() const {
    if (measure_process_cpu_time) return ProcessCPUUsage();
    return ThreadCPUUsage();
  }
```

可以看到 `ThreadTimer` 作为入参被传入 `b->Run` 方法，这里的调用链路是：

```cpp
// b is instance of BenchmarkInstance
State st = b->Run(iters, thread_id, &timer, manager,
                    perf_counters_measurement, profiler_manager_);

State BenchmarkInstance::Run(
    IterationCount iters, int thread_id, internal::ThreadTimer* timer,
    internal::ThreadManager* manager,
    internal::PerfCountersMeasurement* perf_counters_measurement,
    ProfilerManager* profiler_manager) const {
  State st(name_.function_name, iters, args_, thread_id, threads_, timer,
           manager, perf_counters_measurement, profiler_manager);
  benchmark_.Run(st);
  return st;
}
```

在 benchmark::State 中，有根据运行状态操作 ThreadTimer 的接口，会调用：

```cpp
// thread_timer.h

// 启动计时器，记录当前的实时时间和 CPU 时间
void StartTimer() {
  running_ = true;
  start_real_time_ = ChronoClockNow();
  start_cpu_time_ = ReadCpuTimerOfChoice();
}

// 停止计时器，计算从启动到停止所经过的实时时间和 CPU 时间，并将其累加到 real_time_used_ 和 cpu_time_used_ 中
void StopTimer() {
  BM_CHECK(running_);
  running_ = false;
  real_time_used_ += ChronoClockNow() - start_real_time_;
  // Floating point error can result in the subtraction producing a negative
  // time. Guard against that.
  cpu_time_used_ +=
      std::max<double>(ReadCpuTimerOfChoice() - start_cpu_time_, 0);
}
```

以 `ChronoClockNow` 为例，时间是如何被统计的，可以看到`benchmark`为了跨平台，支持了多种时间戳计算方式，在`Linux`环境下，最终调用的就是`std::chrono::high_resolution_clock`:

```cpp
struct ChooseClockType {
#if defined(BENCHMARK_OS_QURT)
  typedef QuRTClock type;
#elif defined(HAVE_STEADY_CLOCK)
  typedef ChooseSteadyClock<>::type type;
#else
  typedef std::chrono::high_resolution_clock type;
#endif
};

inline double ChronoClockNow() {
  typedef ChooseClockType::type ClockType;
  using FpSeconds = std::chrono::duration<double, std::chrono::seconds::period>;
  return FpSeconds(ClockType::now().time_since_epoch()).count();
}
```

那么 `State` 是如何调用的呢，在一个基本的 `benchmark` 里，`State` 类的 `KeepRunning` 方法用于控制循环的执行次数，并且在循环开始和结束时调用 `ThreadTimer` 的 `StartTimer` 和 `StopTimer` 方法：

```cpp
// benchmark.h
inline BENCHMARK_ALWAYS_INLINE bool State::KeepRunning() {
  return KeepRunningInternal(1, /*is_batch=*/false);
}

inline BENCHMARK_ALWAYS_INLINE bool State::KeepRunningInternal(IterationCount n,
                                                               bool is_batch) {
  // total_iterations_ is set to 0 by the constructor, and always set to a
  // nonzero value by StartKepRunning().
  assert(n > 0);
  // n must be 1 unless is_batch is true.
  assert(is_batch || n == 1);
  if (BENCHMARK_BUILTIN_EXPECT(total_iterations_ >= n, true)) {
    total_iterations_ -= n;
    return true;
  }
  if (!started_) {
    StartKeepRunning();
    if (!skipped() && total_iterations_ >= n) {
      total_iterations_ -= n;
      return true;
    }
  }
  // For non-batch runs, total_iterations_ must be 0 by now.
  if (is_batch && total_iterations_ != 0) {
    batch_leftover_ = n - total_iterations_;
    total_iterations_ = 0;
    return true;
  }
  FinishKeepRunning();
  return false;
}

// benchmark.cc
void State::StartKeepRunning() {
  BM_CHECK(!started_ && !finished_);
  started_ = true;
  total_iterations_ = skipped() ? 0 : max_iterations;
  if (BENCHMARK_BUILTIN_EXPECT(profiler_manager_ != nullptr, false)) {
    profiler_manager_->AfterSetupStart();
  }
  manager_->StartStopBarrier();
  if (!skipped()) {
    ResumeTiming();
  }
}

void State::ResumeTiming() {
  BM_CHECK(started_ && !finished_ && !skipped());
  timer_->StartTimer();
  if (perf_counters_measurement_ != nullptr) {
    perf_counters_measurement_->Start();
  }
}

void State::FinishKeepRunning() {
  BM_CHECK(started_ && (!finished_ || skipped()));
  if (!skipped()) {
    PauseTiming();
  }
  // Total iterations has now wrapped around past 0. Fix this.
  total_iterations_ = 0;
  finished_ = true;
  manager_->StartStopBarrier();
  if (BENCHMARK_BUILTIN_EXPECT(profiler_manager_ != nullptr, false)) {
    profiler_manager_->BeforeTeardownStop();
  }
}

void State::PauseTiming() {
  // Add in time accumulated so far
  BM_CHECK(started_ && !finished_ && !skipped());
  timer_->StopTimer();
  if (perf_counters_measurement_ != nullptr) {
    std::vector<std::pair<std::string, double>> measurements;
    if (!perf_counters_measurement_->Stop(measurements)) {
      BM_CHECK(false) << "Perf counters read the value failed.";
    }
    for (const auto& name_and_measurement : measurements) {
      const std::string& name = name_and_measurement.first;
      const double measurement = name_and_measurement.second;
      // Counter was inserted with `kAvgIterations` flag by the constructor.
      assert(counters.find(name) != counters.end());
      counters[name].value += measurement;
    }
  }
}
```

统计到的时间会被累加到 `ThreadManager::Result` 结构体的中。在所有线程执行完毕后，`ThreadManager` 会等待所有线程完成，并汇总每个线程的统计信息。最终，整个基准测试的 CPU 开销就是所有线程的 `cpu_time_used` 之和。