# google benchmark

> [c++性能测试工具：google benchmark入门（一）](https://www.cnblogs.com/apocelipes/p/10348925.html)

## 案例

```cpp
#include <benchmark/benchmark.h>
#include <array>
 
constexpr int len = 6;
 
// constexpr function具有inline属性，你应该把它放在头文件中
constexpr auto my_pow(const int i)
{
    return i * i;
}
 
// 使用operator[]读取元素，依次存入1-6的平方
static void bench_array_operator(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr[0] = my_pow(i);
        arr[1] = my_pow(i+1);
        arr[2] = my_pow(i+2);
        arr[3] = my_pow(i+3);
        arr[4] = my_pow(i+4);
        arr[5] = my_pow(i+5);
    }
}
BENCHMARK(bench_array_operator);
 
// 使用at()读取元素，依次存入1-6的平方
static void bench_array_at(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        arr.at(0) = my_pow(i);
        arr.at(1) = my_pow(i+1);
        arr.at(2) = my_pow(i+2);
        arr.at(3) = my_pow(i+3);
        arr.at(4) = my_pow(i+4);
        arr.at(5) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_at);
 
// std::get<>(array)是一个constexpr function，它会返回容器内元素的引用，并在编译期检查数组的索引是否正确
static void bench_array_get(benchmark::State& state)
{
    std::array<int, len> arr;
    constexpr int i = 1;
    for (auto _: state) {
        std::get<0>(arr) = my_pow(i);
        std::get<1>(arr) = my_pow(i+1);
        std::get<2>(arr) = my_pow(i+2);
        std::get<3>(arr) = my_pow(i+3);
        std::get<4>(arr) = my_pow(i+4);
        std::get<5>(arr) = my_pow(i+5);
    }
}
BENCHMARK(bench_array_get);
 
BENCHMARK_MAIN();
```

可以看到每一个`benchmark`测试用例都是一个类型为`std::function<void(benchmark::State&)>`的函数，其中`benchmark::State&`负责测试的运行及额外参数的传递。

随后我们使用`for (auto _: state) {}`来运行需要测试的内容，`state`会选择合适的次数来运行循环，时间的计算从循环内的语句开始，所以我们可以选择像例子中一样在for循环之外初始化测试环境，然后在循环体内编写需要测试的代码。

测试用例编写完成后我们需要使用`BENCHMARK(<function_name>);`将我们的测试用例注册进`benchmark`，这样程序运行时才会执行我们的测试。

最后是用`BENCHMARK_MAIN();`替代直接编写的main函数，它会处理命令行参数并运行所有注册过的测试用例生成测试结果。

示例中大量使用了`constexpr`，这是为了能在编译期计算出需要的数值避免对测试产生太多噪音。

```shell
g++ -Wall -std=c++14 benchmark_example.cpp -pthread -lbenchmark
```

benchmark需要链接libbenchmark.so，所以需要指定-lbenchmark，此外还需要thread的支持，因为libstdc++不提供thread的底层实现，我们需要pthread。另外不建议使用-lpthread，官方表示会出现兼容问题，在我这测试也会出现链接错误。注意文件名一定要在-lbenchmark前面，否则编译会失败，具体参见：https://github.com/google/benchmark/issues/619

## 传参

```cpp
static void bench_array_ring_insert_int(benchmark::State& state)
{
    auto length = state.range(0);
    auto ring = ArrayRing<int>(length);
    for (auto _: state) {
        for (int i = 1; i <= length; ++i) {
            ring.insert(i);
        }
        state.PauseTiming();
        ring.clear();
        state.ResumeTiming();
    }
}
BENCHMARK(bench_array_ring_insert_int)->Arg(10);
```

* 传递参数使用`BENCHMARK`宏生成的对象的`Arg`方法
* 传递进来的参数会被放入state对象内部存储，通过`range`方法获取，调用时的参数0是传入参数的需要，对应第一个参数
* Arg方法一次只能传递一个参数

```cpp
static void bench_array_ring_insert_int(benchmark::State& state)
{
    auto ring = ArrayRing<int>(state.range(0));
    for (auto _: state) {
        for (int i = 1; i <= state.range(1); ++i) {
            ring.insert(i);
        }
        state.PauseTiming();
        ring.clear();
        state.ResumeTiming();
    }
}
BENCHMARK(bench_array_ring_insert_int)->Args({10, 10});
```

* `Args`方法接受一个vector对象，所以我们可以使用c++11提供的大括号初始化器简化代码，获取参数依然通过state.range方法，1对应传递进来的第二个参数

Arg和Args会将我们的测试用例使用的参数进行注册以便产生用例名/参数的新测试用例，并且返回一个指向BENCHMARK宏生成对象的指针，换句话说，如果我们想要生成仅仅是参数不同的多个测试的话，只需要链式调用Arg和Args即可：

```cpp
BENCHMARK(bench_array_ring_insert_int)->Arg(10)->Arg(100)->Arg(1000);
```

这还不是最优解，我们仍然重复调用了Arg方法，如果我们需要更多用例时就不得不又要做重复劳动了。

对此google benchmark也有解决办法：我们可以使用Range方法来自动生成一定范围内的参数。

```cpp
BENCHMAEK(func)->Range(int64_t start, int64_t limit);
// Range默认除了start和limit，中间的其余参数都会是某一个基底（base）的幂，基地默认为8，所以我们会看到10、64、512、1000
BENCHMARK(bench_array_ring_insert_int)->Range(10, 1000);
// 重新设置基底，通过使用RangeMultiplier方法：
BENCHMARK(bench_array_ring_insert_int)->RangeMultiplier(10)->Range(10, 1000);

// Ragnes可以处理多个参数的情况
BENCHMARK(func)->RangeMultiplier(10)->Ranges({{10, 1000}, {128， 256}});
// 等价于
BENCHMARK(func)->Args({10, 128})
               ->Args({100, 128})
               ->Args({1000, 128})
               ->Args({10, 256})
               ->Args({100, 256})
               ->Args({1000, 256})

```