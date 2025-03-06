# Measure Before Optimization

A simple String Validation Function.

By thinking that `std::set::count` should be cheaper than multiple compare, optimized from `IsValidString` to `IsValidString2`.

After a while, I see `IsValidString2` on perf head map, and then try to figure out why?

Finally, benchmark shows that `IsValidString` is **2.4 times faster** than `IsValidString2`, what a surprise!!!

> [benchmark](https://quick-bench.com/q/0H29pklHRBBc214XUv4X5rOR4oM)

```cpp
#include <string>
#include <set>

int32_t kMetricMaxStrLen = 255;

bool IsValidString(const std::string &s) {
    if (s.length() == 0 || s.length() >= kMetricMaxStrLen) {
        return false;
    }
    for (const auto &c : s) {
        if ('a' <= c && c <= 'z') {
            continue;
        } else if ('A' <= c && c <= 'Z') {
            continue;
        } else if ('.' == c || '_' == c || '-' == c || '/' == c || ':' == c || '%' == c) {
            continue;
        } else if ('0' <= c && c <= '9') {
            continue;
        } else {
            return false;
        }
    }
    return true;
}

static const std::set<char> valid_char = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f',
        'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 'A', 'B',
        'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X',
        'Y', 'Z', '.', '_', '-', '/', ':', '%'};

bool IsValidString2(const std::string &s) {
    if (s.length() == 0 || s.length() >= kMetricMaxStrLen) {
        return false;
    }
    for (const auto &c : s) {
        if (0 == valid_char.count(c)) {
            return false;
        }
    }
    return true;
}


static void StringValidOne(benchmark::State& state) {
  for (auto _ : state) {
    bool ret = IsValidString("qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM1234567890.-_/:%");
    ret = IsValidString("test,myname");
    benchmark::DoNotOptimize(ret);
  }
}
BENCHMARK(StringValidOne);

static void StringValidTwo(benchmark::State& state) {
  for (auto _ : state) {
    bool ret = IsValidString2("qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM1234567890.-_/:%");
    ret = IsValidString2("test,myname");
    benchmark::DoNotOptimize(ret);
  }
}
BENCHMARK(StringValidTwo);

BENCHMARK_MAIN();
```
