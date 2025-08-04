# [Parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack)

在高并发后端服务开发中，代码的性能和灵活性往往决定了系统的上限。C++11引入的参数包（Parameter Pack）特性为我们提供了一种强大的工具，能够显著提升代码的表达能力和运行效率。本文将深入探讨参数包的原理、应用场景和实际案例，帮助开发者在高性能后端服务中充分利用这一特性。

## 基本概念和原理

`Pack (since C++11)` is a C++ entity that defines one of the following:

* a parameter pack
  * template parameter pack
  * function parameter pack
* [lambda init-capture pack](https://en.cppreference.com/w/cpp/language/lambda.html#Lambda_capture) (since C++20)
* [structured binding pack](https://en.cppreference.com/w/cpp/language/structured_binding.html) (since C++26)

**参数包（Parameter Pack）**是C++11引入的变长模板参数特性，允许模板接受任意数量、任意类型的参数。它是C++模板元编程中的一项重要工具，为泛型编程提供了更强大的表达能力。

在语法上，参数包通过省略号（`...`）来表示：

```cpp
template <typename... Args>
void function(Args... args);
```

这里的`Args`是一个**模板参数包**，表示零个或多个类型；而`args`是一个**函数参数包**，表示零个或多个函数参数。

在类模板中，模板参数包必须是模板参数列表中的最后一个参数。而在函数模板中，模板参数包可以出现在列表中靠前的位置，前提是其后面的所有参数都能从函数实参中推导出来，或者有默认参数

```cpp
template<typename U, typename... Ts>    // OK: can deduce U
struct valid;
// template<typename... Ts, typename U> // Error: Ts... not at the end
// struct Invalid;
 
template<typename... Ts, typename U, typename=void>
void valid(U, Ts...);    // OK: can deduce U
// void valid(Ts..., U); // Can't be used: Ts... is a non-deduced context in this position
 
valid(1.0, 1, 2, 3);     // OK: deduces U as double, Ts as {int, int, int}
```

省略号(...)在可变参数模板中有两种用途：

* 省略号出现在形参名字左侧，声明了一个参数包（parameter pack）。使用这个参数包，可以绑定0个或多个模板实参给这个模板参数包。参数包也可以用于非类型的模板参数。
* 省略号出现在包含参数包的表达式的右侧，称作参数包展开（parameter pack expansion），是把这个参数包解开为一组实参，使得在省略号前的整个表达式使用每个被解开的实参完成求值，所有表达式求值结果被逗号分开。

参数包的核心机制是**编译期展开**。编译器会根据调用时提供的实际参数，生成相应的函数实例。这一过程完全在编译期完成，不会带来运行时开销。

参数包展开主要有以下几种方式：

1. **直接展开**：将参数包中的每个元素应用到相同的模式中

```cpp
template <typename... Args>
void printAll(Args... args) {
    (std::cout << ... << args) << std::endl;  // C++17折叠表达式
}
```

2. **递归展开**：通过递归模板实例化展开参数包

```cpp
// 基本情况
void print() {}

// 递归情况
template <typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first;
    print(rest...);
}
```

3. **索引序列展开**：使用整数序列辅助展开

```cpp
template <typename Tuple, std::size_t... Is>
void printTuple(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << std::get<Is>(t) << " "), ...);
}

template <typename... Args>
void printTuple(const std::tuple<Args...>& t) {
    printTuple(t, std::index_sequence_for<Args...>{});
}
```

从编译原理的角度看，参数包处理涉及以下步骤：

1. **模板实例化**：编译器根据调用点生成具体的模板实例
2. **参数推导**：确定每个参数的具体类型
3. **代码生成**：为每个实例生成对应的机器代码

这一过程完全在编译期进行，生成的代码与手写的特化版本相比没有性能差异，但大大提高了代码的灵活性和复用性。

```cpp
#include <iostream>

template<typename... T>
void func(T... args) {}

template <typename... Args>
void printAll(Args... args) {
    (std::cout << ... << args) << std::endl;  // C++17折叠表达式
}

int main() {
    func(1);

    func(1, "hello");

    func(1, "hello", false);

    printAll(1);

    printAll(1, "hello");

    printAll(1, "hello", false);

    return 0;
}
```

以上面代码为例，在 x86-64 clang 19.1.0 下得到的汇编如下：

![param pack expansion example](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CppLearn/param_pack_asm.png)

## Pack expansion