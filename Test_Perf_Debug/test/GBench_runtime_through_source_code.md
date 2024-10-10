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

`BenchmarkFamilies`是一个全局单例，并利用`vector`来保存构建出来的`Benchmark`

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

排除一些分支逻辑和输出逻辑，这里的两个核心函数是：`FindBenchmarksInternal`, `RunBenchmarks`，我们逐个分析：
