# Examples using concept and requires

## 简单例子

> using x86_64 clang 20.1.0 -std=c++2a

利用require和逻辑运算自定义concept

```cpp
// https://godbolt.org/z/8E6oW1a5d
#include <iostream>
#include <concepts>
#include <type_traits>

// 定义一个概念：可加类型
template <typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

// 定义一个概念：可乘类型
template <typename T>
concept Multipliable = requires(T a, T b) {
    { a * b } -> std::same_as<T>;
};

// 定义一个概念：数字类型（支持加法和乘法）
template <typename T>
concept Number = Addable<T> && Multipliable<T>;

void printNumber(Number auto num) {
    std::cout << num << std::endl;
}

int main() {
    printNumber(42);       // OK
    printNumber(3.14);     // OK
    // printNumber("Hello"); // 错误：不满足 Number 概念
    return 0;
}
```

## 实现一个安全的类型转换函数

> https://godbolt.org/z/19jn7dTWs

```cpp
// using x86_64 clang 20.1.0 -std=c++2a
#include <type_traits>
#include <cstdint>
#include <assert.h>

template <typename T> constexpr auto make_false() { return false; }

template <typename Dst, typename Src>
auto safe_cast(const Src &v) -> Dst {
    using namespace std;
    constexpr auto is_same_type = is_same_v<Src, Dst>;
    constexpr auto is_pointer_to_pointer = is_pointer_v<Src> && is_pointer_v<Dst>;
    constexpr auto is_float_to_float = is_floating_point_v<Src> && is_floating_point_v<Dst>; 
    constexpr auto is_number_to_number = is_arithmetic_v<Src> && is_arithmetic_v<Dst>;
    constexpr auto is_intptr_to_ptr = (is_same_v<uintptr_t,Src> || is_same_v<intptr_t,Src>) && is_pointer_v<Dst>;
    constexpr auto is_ptr_to_intptr = is_pointer_v<Src> && (is_same_v<uintptr_t,Dst> || is_same_v<intptr_t,Dst>); 
    
    if constexpr(is_same_type) { 
        return v; 
    } else if constexpr(is_intptr_to_ptr || is_ptr_to_intptr){
        return reinterpret_cast<Dst>(v); 
    } else if constexpr(is_pointer_to_pointer) { 
        assert(dynamic_cast<Dst>(v) != nullptr); 
        return static_cast<Dst>(v); 
    } else if constexpr (is_float_to_float) { 
        auto casted = static_cast<Dst>(v); 
        auto casted_back = static_cast<Src>(v); 
        assert(!isnan(casted_back) && !isinf(casted_back)); 
        return casted; 
    }  else if constexpr (is_number_to_number) { 
        auto casted = static_cast<Dst>(v); 
        auto casted_back = static_cast<Src>(casted); 
        assert(casted == casted_back); 
        return casted; 
    } else {
        static_assert(make_false<Src>(),"CastError");
        return Dst{}; // This can never happen, 
        // the static_assert should have failed 
  }
}

int main() {
    auto x = safe_cast<int>(42.0f); 

    return 0;
}

// 只实现了 is_number_to_number 分支
// int safe_cast<int, float>(float const&):
//         push    rbp
//         mov     rbp, rsp
//         sub     rsp, 32
//         mov     qword ptr [rbp - 8], rdi
//         mov     byte ptr [rbp - 9], 0
//         mov     byte ptr [rbp - 10], 0
//         mov     byte ptr [rbp - 11], 0
//         mov     byte ptr [rbp - 12], 1
//         mov     byte ptr [rbp - 13], 0
//         mov     byte ptr [rbp - 14], 0
//         mov     rax, qword ptr [rbp - 8]
//         cvttss2si       eax, dword ptr [rax]
//         mov     dword ptr [rbp - 20], eax
//         cvtsi2ss        xmm0, dword ptr [rbp - 20]
//         movss   dword ptr [rbp - 24], xmm0
//         cvtsi2ss        xmm0, dword ptr [rbp - 20]
//         ucomiss xmm0, dword ptr [rbp - 24]
//         jne     .LBB1_2
//         jp      .LBB1_2
//         jmp     .LBB1_3
```

## 一个接收字符串字面值的，支持编译时计算hash值的结构

```cpp
constexpr auto hash_func(const char *str) -> size_t {
    // simple implementation for demonstration
    auto sum = size_t{0};
    for (auto ptr = str; *ptr != '\0'; ++ptr)
        sum += *ptr;
    return sum;
}

class PrehashedString {
public:
  // 请注意构造函数中的模板技巧。这迫使PrehashedString只接受编译时字符串字面值。这样做的原因是PrehashedString类不拥有const char* ptr，因此我们只能在编译时使用它创建的字符串字面值
  template <size_t N>
  constexpr PrehashedString(const char(&str)[N])
      : hash_{hash_function(&str[0])}, size_{N - 1},
      // The subtraction is to avoid null at end
        strptr_{&str[0]} {}
  auto operator==(const PrehashedString& s) const {
    return
      size_ == s.size_ &&
      std::equal(c_str(), c_str() + size_, s.c_str());
  }
  auto operator!=(const PrehashedString& s) const {
    return !(*this == s); }
  constexpr auto size()const{ return size_; }
  constexpr auto get_hash()const{ return hash_; }
  constexpr auto c_str()const->const char*{ return strptr_; }
private:
  size_t hash_{};
  size_t size_{};
  const char* strptr_{nullptr};
};
namespace std {
template <>
struct hash<PrehashedString> {
  constexpr auto operator()(const PrehashedString& s) const {
    return s.get_hash();
  }
};
} // namespace std 

```
