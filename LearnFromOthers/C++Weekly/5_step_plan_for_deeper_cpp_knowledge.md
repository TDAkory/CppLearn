# [Your 5 Step Plan For Deeper C++ Knowledge](https://www.youtube.com/watch?v=287_oG4CNMc)

> https://github.com/lefticus/cpp_weekly/issues/176

1. Create a tool that helps you understand object lifetime

```cpp
#include <cstdio>

struct Lifetime {
    Lifetime() noexcept { puts("Lifetime() [default constructor]"); }
    Lifetime(const Lifetime &) noexcept {
        puts("Lifetime(const Lifetime &) [copy constructor]");
    }
    Lifetime(Lifetime &&) noexcept {
        puts("Lifetime(Lifetime &&) [move constructor]");
    }
    ~Lifetime() noexcept { puts("~Lifetime() [destructor]"); }
    Lifetime &operator=(const Lifetime &) noexcept {
        puts("Lifetime &operator=(const Lifetime &) [copy assignment]");
    }
    Lifetime &operator=(Lifetime &&) noexcept {
        puts("Lifetime &operator=(Lifetime &&) [move assignment]");
    }
};
```

2. Study The Lambda!
3. Create a std::function implmentation
   1. lambdas
   2. free functions
   3. member functions
   4. static member function
4. Make std::function constexpr
5. Implement Small Function Optimization