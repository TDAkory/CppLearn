# [Compile-Time Validation in C++ Programming](https://www.youtube.com/watch?v=jDn0rxWr0RY&list=PLHTh1InhhwT6U7t1yP2K8AtTEKmcM3XU_&index=9)

- [Compile-time validation](https://blog.andreiavram.ro/cpp-compilation-time-validation/)

* memory safety != software safety

```cpp
// runtime error report
void foo() {
    auto error = detect_error();
    if (error) {
        report_error();
    }
}

// runtime error reporting can be used with compile-time error detection
void foo() {
    constexpr auto error = detect_error();
    if constexpr (error) {
        report_error();
    }
}

// error reporting with static_assert, err msg must be a string literal
void foo() {
    constexpr auto error = detect_error();
    static_assert(!error, "err msg");
}

// generate custom error messages at compile-time for better diagnostics
struct custom_error {};

void foo() {
    constexpr auto error = std::optional(custom_error{});
    if consexpr (error) {
        report_error<*error>();
    }
}

template<auto error>
constexpr auto report_error() {
    static_assert(sizeof(error) == 0); // size can never be 0
}
```