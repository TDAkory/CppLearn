# C++高性能编程

- [C++高性能编程](#c高性能编程)
  - [2. 基本C++技术](#2-基本c技术)
    - [`auto`自动类型推到](#auto自动类型推到)
    - [移动语义](#移动语义)
    - [设计带有错误处理的接口](#设计带有错误处理的接口)
    - [函数对象和lambda表达式](#函数对象和lambda表达式)
  - [3. 分析和测量性能](#3-分析和测量性能)
  - [4. 数据结构](#4-数据结构)
  - [5. 算法](#5-算法)
    - [定义在中的标准算法通常具有如下特性：](#定义在中的标准算法通常具有如下特性)
      - [算法不会改变容器的大小(Functions from  can only modify the elements in a specified range)。](#算法不会改变容器的大小functions-from--can-only-modify-the-elements-in-a-specified-range)
      - [带有输出的算法需要预选分配目标容器空间(Algorithms with output require allocated data)](#带有输出的算法需要预选分配目标容器空间algorithms-with-output-require-allocated-data)
      - [算法默认使用`operator==()`和`operator()<`](#算法默认使用operator和operator)
      - [受限算法使用投影](#受限算法使用投影)
      - [算法要求`Move`操作不能抛出异常](#算法要求move操作不能抛出异常)
      - [算法具有复杂性保证](#算法具有复杂性保证)
      - [算法的性能与C库函数的等价一样好](#算法的性能与c库函数的等价一样好)
    - [最佳实践](#最佳实践)
  - [6. 范围和视图](#6-范围和视图)
    - [从Ranges库理解视图](#从ranges库理解视图)
      - [视图是可组合的](#视图是可组合的)
      - [视图是具有复杂性保证的非拥有范围](#视图是具有复杂性保证的非拥有范围)
      - [视图不会改变底层容器](#视图不会改变底层容器)
      - [视图可以实体化为容器](#视图可以实体化为容器)
      - [视图是延迟计算的（惰性评估）](#视图是延迟计算的惰性评估)
    - [标准库的视图](#标准库的视图)
      - [Range views](#range-views)
      - [重新审视 std::string\_view 和 std::span](#重新审视-stdstring_view-和-stdspan)
  - [7.内存管理](#7内存管理)
    - [内存对齐](#内存对齐)
    - [内存所有权](#内存所有权)
      - [隐式处理资源](#隐式处理资源)
    - [小对象优化](#小对象优化)
    - [自定义内存管理](#自定义内存管理)
  - [8. 编译时编程](#8-编译时编程)
    - [模版元编程](#模版元编程)
    - [类型特征（`<type_traits>`）](#类型特征type_traits)
    - [常量表达式](#常量表达式)
    - [`concept`和`requires`](#concept和requires)
  - [9.基本实用程序](#9基本实用程序)
    - [`std::optional<T>`](#stdoptionalt)
    - [固定大小异构集合](#固定大小异构集合)
    - [动态大小的异构集合](#动态大小的异构集合)
      - [std::variant 的异常安全性](#stdvariant-的异常安全性)
      - [访问变体](#访问变体)
      - [全局函数 `std::get()`](#全局函数-stdget)
  - [10. 代理对象和延迟评估](#10-代理对象和延迟评估)
    - [使用代理对象的一些例子](#使用代理对象的一些例子)
  - [11. 并发](#11-并发)
    - [并发和并行](#并发和并行)
      - [共享内存](#共享内存)
      - [数据竞争](#数据竞争)
      - [互斥锁和临界区](#互斥锁和临界区)
      - [死锁](#死锁)
    - [并发编程](#并发编程)
      - [C++20中的同步原语](#c20中的同步原语)
      - [C++中的原子支持](#c中的原子支持)
      - [内存模型和无锁编程](#内存模型和无锁编程)
    - [性能指南](#性能指南)
    - [伪共享](#伪共享)
  - [12. 协程和惰性生成器（Coroutine \& Lzay Generator）](#12-协程和惰性生成器coroutine--lzay-generator)
    - [Abstraction](#abstraction)
    - [有栈协程和无栈协程](#有栈协程和无栈协程)
      - [Python：`greenlet`库](#pythongreenlet库)
      - [Lua：`coroutine`库](#luacoroutine库)
      - [（3）Go：Goroutine（特殊的有栈协程）](#3gogoroutine特殊的有栈协程)
    - [2. 无栈协程的实践](#2-无栈协程的实践)
      - [（1）C++20：`std::coroutine`](#1c20stdcoroutine)
      - [（2）C#：`async/await`](#2casyncawait)
      - [（3）JavaScript：`async/await`](#3javascriptasyncawait)
      - [（4）Python：`asyncio`（原生无栈协程）](#4pythonasyncio原生无栈协程)
    - [如何选择：场景决定类型](#如何选择场景决定类型)
  - [13. 使用协程进行异步编程](#13-使用协程进行异步编程)
  - [14. 并行算法](#14-并行算法)
    - [算法并行化的一般思路](#算法并行化的一般思路)
      - [方案1：循环并行（最常用，适合无依赖循环）](#方案1循环并行最常用适合无依赖循环)
      - [方案2：分治并行（适合递归算法，如排序、查找）](#方案2分治并行适合递归算法如排序查找)
      - [方案3：数据并行（适合大规模数据处理，如滤波、统计）](#方案3数据并行适合大规模数据处理如滤波统计)
      - [并行化避坑指南（常见问题与解决方案）](#并行化避坑指南常见问题与解决方案)
    - [并行标准库算法](#并行标准库算法)
    - [基于索引的for\_each](#基于索引的for_each)
  - [15. 其他书籍推荐](#15-其他书籍推荐)


> [C++ 高性能编程（全）](https://www.cnblogs.com/apachecn/p/18172912)
> [C++ High Performance, Second Edition: Master the art of optimizing the functioning of your C++ code](https://www.amazon.com/dp/1839216549)

**零开销原则**：你不使用的东西，你就不需要付费；你使用的东西，你无法手工编码得更好

一些 C++ 的旧但强大的特性，这些特性与健壮性有关，而不是性能，很容易被忽视：值语义、const正确性、所有权、确定性销毁和引用。

> 在内部，引用是一个不允许为空或重新指向的指针；因此，当将其传递给函数时不涉及复制。

**C++的缺点**：长时间的编译时间和导入库的复杂性（直到 C++20，C++一直依赖于一个过时的导入系统）；缺乏提供的库（C++提供的几乎只是最基本的算法、线程，以及从 C++17 开始的文件系统处理，图形、用户界面、网络、线程、资源处理等只能依赖外部库）

## 2. 基本C++技术

### `auto`自动类型推到

**在函数签名中使用 `auto`**

| 显式类型的传统语法                       | 使用 `auto` 的新语法                |
| ---------------------------------------- | ----------------------------------- |
| `int val() const { return m_; }`         | `auto val() const { return m_; }`   |
| `const int& cref() const { return m_; }` | `auto& cref() const { return m_; }` |
| `int& mref() { return m_; }`             | `auto& mref() { return m_; }`       |

**使用 `decltype(auto)` 进行返回类型转发**

```cpp
int val_wrapper() { return val(); }    // 返回 int
int& mref_wrapper() { return mref(); } // 返回 int&

auto val_wrapper() { return val(); }   // 返回 int
auto mref_wrapper() { return mref(); } // 也返回 int

decltype(auto) val_wrapper() { return val(); }   // 返回 int
decltype(auto) mref_wrapper() { return mref(); } // 返回 int&
```

**使用 `auto` 声明变量**

```cpp
auto i = 0;
auto x = Foo{};
auto y = create_object();
auto z = std::mutex{};     // OK si
```

在 `C++17` 中引入了保证的拷贝省略，语句`auto x = Foo{}`与`Foo x{}`是相同的；也就是说，语言保证在这种情况下没有需要移动或复制的临时对象。

使用`auto`定义变量的一个很大的优势是，永远不会留下未初始化的变量，因为`auto x;`不会编译。

`const auto&`表示，具有绑定到任何东西的能力。原始对象永远不会通过这样的引用发生变异。是潜在昂贵的对象的默认选择。

`auto&`来表示可变引用。只有在打算更改引用的对象时才使用可变引用。

`auto&&`被称为转发引用（也称为通用引用）。它可以绑定到任何东西，这对某些情况很有用。转发引用将像`const`引用一样，延长临时对象的生命周期。但与`const`引用相反，`auto&&`允许我们改变它引用的对象，包括临时对象。

便于使用的实践建议

* 基本类型和小的非基本类型：使用 const auto。
* 潜在昂贵的大型类型：使用 const auto&。
* 需要可变引用：使用 auto&。
* 转发代码：使用 auto&&。

[指针的const传播](../../Basic_Concept/C++_Advanced/50_propagate_const.md)

### 移动语义

只有在对象类型拥有某种资源（最常见的情况是堆分配的内存）时，移动对象才有意义。如果所有数据都包含在对象内部，移动对象的最有效方式就是简单地复制它。

**五法则**，**三法则**、移动构造、移动赋值。

C++11引入移动语义之前，是**三法则**，拷贝构造、拷贝赋值、析构。

相对于拷贝，移动函数不会分配内存或执行可能引发异常的操作，通常可以标记为 `noexcept`

不要忘记将您的移动构造函数和移动赋值运算符标记为noexcept（除非它们可能抛出异常）。不标记它们为noexcept会阻止标准库容器和算法在某些条件下使用它们，而是转而使用常规的复制/赋值。详见[vector和性能退化](../../Basic_Concept/Things_About/Things_about_vector_pessimization.md)

那么，编译器何时允许移动对象而不是复制呢？简短的答案是，当对象可以被归类为 rvalue 时，编译器会移动对象。 rvalue 本质是不与命名变量绑定的对象：直接来自函数、`std::move`显示声明

```cpp
// https://godbolt.org/z/Gs8aMaxjf
#include <iostream>
 
class Button { 
public: 
  Button() {} 
  auto set_title(const std::string& s) { 
    std::cout << "Copy assign" << std::endl;
    title_ = s; 
  } 
  auto set_title(std::string&& s) { 
    std::cout << "Move assign" << std::endl;
    title_ = std::move(s); 
  } 
  std::string title_; 
}; 

auto get_ok() {
  return std::string("OK");
}
auto button = Button{}; 
 
int main()
{
    auto str = std::string{"OK"};
    button.set_title(str);              // copy-assigned 
    button.set_title(std::move(str));   // move-assigned 
    button.set_title(get_ok());        // move-assigned 
    {
        auto str = get_ok();
        button.set_title(str);             // copy-assigned 
    }
    {
        const auto str = get_ok();
        button.set_title(std::move(str));  // copy-assigned 
    }
}
```

得到的结果

```shell
Copy assign
Move assign
Move assign
Copy assign
Copy assign
```

如果我们不声明任何自定义复制构造函数/复制赋值或析构函数，移动构造函数/移动赋值将被隐式声明。

* 零规则：尽量避免显式编写复制/移动构造函数、赋值运算符和析构函数。如果需要，使用 default 或完全不定义这些函数。
* 空析构函数：避免使用空析构函数，因为它可能会阻止编译器优化。使用默认析构函数或完全不定义析构函数，以便在应用程序中获得更好的性能。

```cpp
#include <algorithm>

struct Point {
    int x_, y_;
    ~Point() {}  // 空析构函数，不推荐使用！
};

auto copy(Point* src, Point* dst) {
    std::copy(src, src + 64, dst);
}

struct Point2 {
    int x_, y_;
    ~Point2() = default;
};

auto copy2(Point2* src, Point2* dst) {
    std::copy(src, src + 64, dst);
}
```

在🆕6-64 GCC 7.1 -O2下得到的汇编：由于 `Point2` 类使用了默认析构函数（或没有析构函数），编译器可以将其视为平凡类型（`trivially copyable`），因此可以使用 `memmove` 进行优化。`memmove` 是一个高效的内存复制函数，通常比手动循环复制更快。

```asm
copy(Point*, Point*):
        xor     eax, eax                    ; 将 eax 寄存器清零，用于初始化循环计数器
.L2:
        mov     rdx, QWORD PTR [rdi+rax]    ; 从源地址 rdi 加上偏移量 rax 处加载8字节（一个 Point 对象）到 rdx 寄存器
        mov     QWORD PTR [rsi+rax], rdx    ; rdx 寄存器的内容存储到目标地址 rsi 加上偏移量 rax 处
        add     rax, 8                      ; 将循环计数器 rax 增加 8 字节（一个 Point 对象的大小）
        cmp     rax, 512                    ; 比较 rax 和 512（即 64 个 Point 对象的总大小）
        jne     .L2                         ; 如果不相等，跳转回循环标签 .L2 继续复制
        rep ret
copy2(Point2*, Point2*):
        mov     rax, rdi                    ; 将源地址 rdi 保存到 rax 寄存器
        mov     edx, 512                    ; 复制的字节数（512 字节）加载到 edx 寄存器
        mov     rdi, rsi                    ; 将目标地址 rsi 加载到 rdi 寄存器
        mov     rsi, rax                    ; 将源地址 rax 加载到 rsi 寄存器
        jmp     memmove                     ; 跳转到 memmove 函数，使用 memmove 进行优化的内存复制
```

可以向类的成员函数添加`&&`修饰符，就像可以向成员函数应用`const`修饰符一样。与`const`修饰符一样，具有`&&`修饰符的成员函数只有在对象是右值时才会被重载解析考虑:

```cpp
#include <memory>

struct Foo { 
  auto func() && {} 
}; 
auto a = Foo{}; 
a.func();            // Doesn't compile, 'a' is not an rvalue 
std::move(a).func(); // Compiles 
Foo{}.func();        // Compiles 
```

函数返回值不要写`std::move()`，会阻止编译器使用返回值优化（RVO），`RVO`会完全省略了对象的复制，比移动更有效。

### 设计带有错误处理的接口

- **错误类型区分**：需区分编程错误（如违反函数前置条件）和运行时错误，运行时错误又可分为可恢复和不可恢复的。不可恢复错误（如堆栈溢出）常导致程序终止；部分错误在不同应用场景下恢复性不同。
- **标准库行为**：C++ 标准库在内存耗尽时抛出 `std::bad_alloc` 异常，但内存耗尽通常不可恢复。

**设计契约**：契约是调用者与被调用者间的规则，部分无法用 C++ 类型系统表达：

* **前置条件**：规定调用者责任，如调用 `std::vector::pop_back()` 时向量不能为空。
* **后置条件**：规定函数返回时的职责，如 `std::list::sort()` 后元素应升序排列。
* **不变量**：应始终成立的条件，包括循环不变量和类不变量。类不变量定义对象有效状态，如 `std::vector` 的 `size() <= capacity()`。

**类不变量**：定义对象有效状态，指定类内部数据成员关系。有助于设计高内聚、少状态的类，使类更易用、易发现错误

```cpp
// 类不变量应在类的构造函数、析构函数和成员函数中得到维护。例如
struct Widget {
    Widget() {
        // Initialize object…
        // Check class invariant
    }
    ~Widget() {
        // Check class invariant
        // Destroy object…
    }
    auto some_func() {
        // Check precondition (including class invariant)
        // Do the actual work…
        // Check postcondition (including class invariant)
    }
};
```

**合同维护**: 合同是您设计和实现的 API 的一部分，常见的维护方式包括：使用 Boost.Contract 库；记录合同（运行时不检查，文档易过时）；使用 `static_assert()` 和 `assert()` 宏；构建自定义库。

`static_assert()` 用于编译时验证，失败导致编译错误；`assert()` 用于运行时检查，调试和测试时启用，发布模式可通过定义 `NDEBUG` 禁用。

**错误处理策略**：

* 编程错误
  * **处理原则**：使用断言让开发者意识到问题，无需用异常或错误代码处理。
  * **断言作用**：明确代码作者假设，限制需处理情况，方便团队协作。
  * **断言失败处理**：检查代码和断言，可能是代码错误、断言错误或两者皆错，需相应修复。
  * **性能影响**：运行时断言可能降低测试构建性能，但发布版本应忽略。
* 可恢复的运行时错误
  * **处理目的**：将错误传递到可恢复有效状态的代码处。
  * **信号方式**：可选择 C++ 异常、错误代码、`std::optional`、`std::pair`、`boost::outcome` 或 `std::experimental::expected`。

**异常处理**：

* **使用场景**：C++ 标准错误处理机制，构造函数失败时只能用异常发出错误。
* **`noexcept` 标记**：标记函数不抛出异常，编译器不验证，从其抛出异常会调用 `std::terminate()`，有时可生成更快代码。
* **强异常安全性**：函数应像事务一样，要么提交所有状态更改，要么完全回滚。可使用复制和交换惯用法实现，如在临时副本上执行可能抛异常操作，再用非抛出 `swap()` 修改对象状态。
* **自动释放**：C++ 对象销毁可预测，如 `std::scoped_lock` 退出作用域时自动解锁互斥量。

放弃异常的两个主要原因：

* 即使不抛出异常，二进制程序的大小也会增加。尽管这通常不是问题，但它并不遵循零开销原则，因为我们为我们不使用的东西付费。
* 抛出和捕获异常相对昂贵。抛出和捕获异常的运行时成本是不确定的。这使得异常在具有硬实时要求的情况下不适用。在这种情况下，其他替代方案，如返回带有返回值和错误代码的std::pair可能更好。

### 函数对象和lambda表达式

Lambda 表达式生成函数对象，而函数对象是具有调用运算符 operator() 的类的实例。要理解 Lambda 表达式的组成，可以将其视为具有限制的常规类。具体来说：

* 该类只包含一个成员函数：operator()。
* 捕获子句是类的成员变量和其构造函数的组合。

通过值捕获的 lambda

```cpp
auto is_above = x { return y > x;};

// 等价

class IsAbove {
public: 
    IsAbove(int x) : x{x} {} 
    auto operator()(int y) const {   return y > x; }
private: 
    int x{}; // Value 
};
```

通过引用捕获的 lambda

```cpp
auto is_above = &x { return y > x;};

// 等价

class IsAbove {
public: 
    IsAbove(int& x) : x{x} {} 
    auto operator()(int y) const {   return y > x; }
private: 
    int& x; // Reference 
};
```

由于 lambda 的工作方式就像一个具有成员变量的类，它也可以改变它们。然而，lambda 的函数调用运算符默认为const，因此我们需要使用mutable关键字明确指定 lambda 可以改变其成员:

```cpp
auto counter_func = [counter = 1]() mutable {
  std::cout << counter++;
};
```

自 C++20 以来，没有捕获的 lambda 是可默认构造和可赋值的。通过使用decltype，现在可以轻松构造具有相同类型的不同 lambda 对象：

```cpp
auto x = [] {};   // A lambda without captures
auto y = x;       // Assignable
decltype(y) z;    // Default-constructible
static_assert(std::is_same_v<decltype(x), decltype(y)>); // passes
static_assert(std::is_same_v<decltype(x), decltype(z)>); // passes 
```

这仅适用于没有捕获的 lambda。具有捕获的 lambda 有它们自己的唯一类型。即使两个具有捕获的 lambda 函数是彼此的克隆，它们仍然具有自己的唯一类型。因此，不可能将一个具有捕获的 lambda 分配给另一个 lambda。 

即使每个有状态的 lambda 都有其自己独特的类型，一个std::function类型可以包装共享相同签名（返回类型和参数）的 lambda

```cpp
class Button {
public: 
  Button(std::function<void(void)> click) : handler_{click} {} 
  auto on_click() const { handler_(); } 
private: 
  std::function<void(void)> handler_{};
}; 

auto create_buttons () { 
  auto beep = Button([counter = 0]() mutable {  
    std::cout << "Beep:" << counter << "! "; 
    ++counter; 
  }); 
  auto bop = Button([] { std::cout << "Bop. "; }); 
  auto silent = Button([] {});
  return std::vector<Button>{beep, bop, silent}; 
} 
```

`std::function`的一些性能损失：阻止内联优化、捕获变量的动态分配内存（如果将 `std::function` 分配给带有捕获变量/引用的 Lambda，那么 `std::function` 在大多数情况下将使用堆分配的内存来存储捕获的变量。如果捕获变量的大小低于某个阈值，一些 `std::function` 的实现将不分配额外的内存）、额外的运行时计算

为了优化 `std::function` 的性能，可以考虑以下几点：

* 优先使用函数指针或引用：仅在需要封装回调函数或存储不同类型可调用对象时使用 `std::function`。
* 使用 `std::move` 转移所有权：在从一个容器移动到另一个容器时，使用 `std::move` 来减少不必要的复制开销。
* 避免频繁复制 `std::function` 对象：尤其是那些封装了大量状态的可调用对象。
* 编译器优化：确保在编译时打开适当的优化级别，例如使用 `-O2` 或 `-O3`

通用lambda：

```cpp
auto lambda = [](auto x, auto y) {
    return x + y;
};

// 通用 Lambda 的行为类似于模板函数。编译器会为每种不同的参数类型生成对应的实例

// 编译器会将其转换为类似以下的代码：
struct AnonymousLambda {
    template<typename T1, typename T2>
    auto operator()(T1 x, T2 y) const {
        return x + y;
    }
};

// C++20 进一步扩展了 Lambda 的功能，允许显式使用模板语法定义 Lambda。例如：
auto lambda = []<typename T>(T x, T y) {
    return x + y;
};
```

## 3. 分析和测量性能

**摊销时间复杂度**：摊销时间复杂度关注的是一系列操作的总时间，然后将其平均分配到每个操作上。即使某些操作的时间复杂度较高，只要这些操作的发生频率较低，整体平均性能仍然可以很好。

**如何优化**：明确目标、测量、找出瓶颈、做出合理猜测、优化、评估、重构

**性能特性**：延迟/响应时间、吞吐量、IO bound、CPU bound、功耗

**插桩分析**：向程序中插入代码以便分析，以收集关于每个函数被执行频率的信息。

**采样分析**：采样分析器通过在均匀间隔（通常为每 10 毫秒）查看运行程序的状态来创建概要。

在概念上，采样分析器以均匀的时间间隔存储调用堆栈的样本。它检测当前在 CPU 上运行的内容。纯采样分析器通常只检测当前在运行状态的线程中执行的函数，因为休眠线程不会被调度到 CPU 上。这意味着如果一个函数正在等待导致线程休眠的锁，那么这段时间不会显示在时间概要中。这很重要，因为您的瓶颈可能是由线程同步引起的，这可能对采样分析器是不可见的。

**MicroBenchmark（微基准测试）** ：找到需要调整的热点，最好使用分析器；将其与其余代码分离并创建一个孤立的微基准测试；优化微基准测试。使用基准测试框架在优化过程中测试和评估代码；将新优化的代码集成到程序中，然后重新测量，看看当代码在更大的上下文中运行时，优化是否相关。

**MicroBenchmark的陷阱**：编译器可能会以不同于在完整程序中优化的方式来优化孤立的代码；在基准测试中未使用的返回值可能会使编译器删除我们试图测量的函数；在微基准测试中提供的静态测试数据可能会使编译器在优化代码时获得不切实际的优势（比如确定的循环次数导致向量化优化）

## 4. 数据结构

缓存、缓存延迟、时间局部性、空间局部性、缓存抖动

**序列容器**

`std::vector`使用`std::move_if_noexcept`来确定对象是应该被复制还是移动。

作为动态大小向量的替代，标准库还提供了一个名为`std::array`的固定大小版本，它通过使用堆栈而不是自由存储来管理其元素。数组的大小是在编译时指定的模板参数，这意味着大小和类型元素成为具体类型的一部分。

`std::deque`通常实现为一组固定大小的数组，这使得可以在常数时间内通过它们的索引访问元素。

`std::list`是一个双向链表，意味着每个元素都有一个指向下一个元素和一个指向前一个元素的链接。这使得可以向前和向后遍历列表。还有一个名为`std::forward_list`的单向链表。

**关联容器**

有序关联容器：基于树，std::set、std::map、std::multiset和std::multimap。

无序关联容器：基于哈希表，std::unordered_set、std::unordered_map、std::unordered_multiset和std::unordered_multimap

**容器适配器**

标准库中有三种容器适配器：std::stack、std::queue和std::priority_queue。容器适配器与序列容器和关联容器非常不同，因为它们代表可以由底层序列容器实现的抽象数据类型。例如，堆栈是一个后进先出（LIFO）数据结构，支持在堆栈顶部进行推送和弹出，可以使用vector、list、deque或任何其他支持back()、push_back()和pop_back()的自定义序列容器来实现。队列也是如此，它是一个先进先出（FIFO）数据结构，以及priority_queue。

**视图**

C++17 中的`std::string_view`和 C++20 中引入的`std::span`。

`std::span`指向的内存是可变的，而`std::string_view`总是指向常量内存。`std::string_view`还包含特定于字符串的函数，如`hash()`和`substr()`，这自然不是`std::span`的一部分。最后，在`std::span`中没有`compare()`函数，因此不可能直接在`std::span`对象上使用比较运算符。

**性能考虑**

选择合适的容器、使用合适的API

**并行数组**：（Parallel Arrays）是指一组具有相同长度的数组，这些数组中的元素在相同索引位置上存在逻辑关联。也就是说，每个数组代表对象的一个不同属性，而相同索引处的元素组合起来描述一个完整的实体。

优势

* 简单性：实现起来非常直观和简单，不需要定义复杂的类或数据结构。对于简单的应用场景，使用并行数组可以快速地存储和访问相关数据。
* 内存效率：在某些情况下，并行数组可以比使用对象数组更节省内存。因为对象数组通常会包含额外的元数据（如对象头），而并行数组只存储实际的数据。
* 访问速度：由于数组是连续存储的，对并行数组的随机访问速度较快。可以直接通过索引访问每个数组中的元素，时间复杂度为O(1)

劣势

* 可维护性差：随着数据规模的增大和需求的变化，并行数组的代码会变得难以维护。如果需要添加新的属性，就需要同时修改多个数组，容易引入错误。
* 缺乏封装性：并行数组没有将相关的数据封装在一起，数据的逻辑关系不够清晰。这使得代码的可读性和可理解性降低，不利于团队协作和代码的长期维护。
* 容易出错：由于需要手动管理多个数组的索引，容易出现索引不一致的问题。例如，在删除或插入元素时，如果只修改了部分数组的索引，就会导致数据不一致。
* 扩展性有限：当数据的结构变得复杂时，并行数组很难进行扩展。例如，如果需要处理嵌套的数据结构或复杂的关系，使用并行数组会变得非常困难。

## 5. 算法

C++20 通过引入 Ranges 库和 C++Concept 的语言特性对算法库进行了重大改变。

**迭代器和范围**

迭代器是数据结构和算法之间的粘合剂。迭代器抽象根本不是 C++独有的概念，而是存在于大多数编程语言中。C++实现迭代器概念的不同之处在于，C++模仿了原始内存指针的语法。

范围是指我们在引用一系列元素时使用的迭代器-哨兵对的替代品。`<range>`头文件包含了定义不同种类范围要求的多个概念，例如input_range，random_access_range等等。如下 Concept 约束意味着任何暴露begin()和end()函数的类型都被认为是范围

```cpp
template<class T>
concept range = requires(T& t) {
  ranges::begin(t);
  ranges::end(t);
}; 
```

为了适配算法场景，迭代器被设计了六种类型：

* `std::input_iterator`：支持只读和向前移动（一次）。如`std::count()`可以使用输入迭代器

* `std::output_iterator`：支持只写和向前移动（一次）。请注意，输出迭代器只能写入，不能读取。`std::ostream_iterator`是输出迭代器的一个例子。

* `std::forward_iterator`：支持读取，写入和向前移动。当前位置的值可以多次读取或写入。

* `std::bidirectional_iterator`：支持读取，写入，向前移动和向后移动。

* `std::random_access_iterator`：支持读取，写入，向前移动，向后移动和在常数时间内跳转到任意位置。

* `std::contiguous_iterator`：与随机访问迭代器相同，但也保证底层数据是连续的内存块，例如`std::string`，`std::vector`，`std::array`，`std::span`和（很少使用的）`std::valarray`

### 定义在<algorithm>中的标准算法通常具有如下特性：

#### 算法不会改变容器的大小(Functions from <algorithm> can only modify the elements in a specified range)。

`std::remove()`或`std::unique()`实际上并不会从容器中删除元素（尽管它们的名字是这样）。相反，它们将应该保留的元素移动到容器的前面，然后返回一个标记，定义了元素的有效范围的新结尾

#### 带有输出的算法需要预选分配目标容器空间(Algorithms with output require allocated data)

```cpp
const auto square_func = [](int x) { return x * x; };
const auto v = std::vector{1, 2, 3, 4};
auto squared = std::vector<int>{};
std::ranges::transform(v, squared.begin(), square_func); 
```

这段代码在逻辑上存在一个潜在问题，具体如下：`std::ranges::transform` 是 C++20 引入的算法，用于对一个范围内的元素进行变换操作，并将结果存储到另一个范围中。它的基本用法是：`std::ranges::transform(input_range, output_iterator, unary_op);`

`squared` 被初始化为一个空的 `std::vector<int>`，因此 `squared.begin()` 是一个无效的迭代器。`std::ranges::transform` 会尝试将结果存储到 `squared` 中，但由于 `squared` 的大小为 0，这会导致未定义行为（Undefined Behavior，UB）。具体表现可能是程序崩溃、数据损坏或其他不可预测的行为。

要解决这个问题，可以采用以下方法之一：

```cpp
// 在调用 `std::ranges::transform` 之前，为 `squared` 分配足够的空间，使其大小与输入范围 `v` 一致
squared.resize(v.size());
std::ranges::transform(v, squared.begin(), square_func);
```

或者

```cpp
// 使用 `std::back_inserter` 来自动扩展 `squared` 的大小：
std::ranges::transform(v, std::back_inserter(squared), square_func);
```

#### 算法默认使用`operator==()`和`operator()<`

#### 受限算法使用投影

在 C++ 中，**受限算法（Constrained Algorithms）** 和 **投影（Projection）** 是 C++20 标准引入的两个重要概念，它们与范围（Ranges）库紧密相关，用于提高算法的灵活性和表达能力。

受限算法是 C++20 中对标准算法库的改进。在 C++20 之前，标准算法库中的算法（如 `std::sort`、`std::find` 等）通常只接受特定类型的迭代器或容器。然而，C++20 引入了 **范围（Ranges）**，允许算法直接作用于范围对象，而不仅仅是迭代器对。受限算法通过模板约束（Concepts）来限制算法的输入，使其能够更自然地与范围库结合。

以下是一个使用受限算法的示例：

```cpp
// `std::ranges::sort` 是一个受限算法，它直接作用于 `std::vector<int>` 的范围对象 `v`，而不需要显式地传递迭代器对
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
    std::ranges::sort(v); // 受限算法，直接作用于范围
    for (const auto& x : v) {
        std::cout << x << " ";
    }
    return 0;
}
```

投影是 C++20 范围库中引入的一个概念，用于在算法中对范围的元素进行转换或映射。投影允许算法在处理元素之前，先对元素进行某种操作，从而实现更灵活的算法行为。

投影通常通过一个可调用对象（如函数、lambda 表达式或函数对象）来实现。在调用算法时，可以通过 `std::ranges::views::transform` 或直接传递投影函数来指定投影操作。

以下是一个使用投影的示例：

```cpp
// `std::views::transform` 是一个投影操作，它将 `v` 中的每个字符串映射为其长度。投影的结果是一个新的范围对象 `lengths`，包含每个字符串的长度
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<std::string> v = {"apple", "banana", "cherry"};
    auto lengths = v | std::views::transform([](const std::string& s) { return s.length(); });
    for (const auto& len : lengths) {
        std::cout << len << " ";
    }
    return 0;
}
```

投影也可以直接在受限算法中使用。例如，`std::ranges::max` 算法可以通过投影来指定比较的依据：

```cpp
// `std::ranges::max` 的第三个参数是一个投影函数，用于指定比较的依据（即字符串的长度）
#include <ranges>
#include <vector>
#include <iostream>

int main() {
    std::vector<std::string> v = {"apple", "banana", "cherry"};
    auto longest = std::ranges::max(v, {}, [](const std::string& s) { return s.length(); });
    std::cout << "Longest string: " << longest << std::endl;
    return 0;
}
```

- **受限算法**：通过模板约束和范围支持，使算法更加灵活和类型安全。
- **投影**：允许在算法中对范围的元素进行转换或映射，从而实现更灵活的算法行为。
- **结合使用**：受限算法和投影可以结合使用，以实现更强大的功能，提高代码的可读性和灵活性。

#### 算法要求`Move`操作不能抛出异常

#### 算法具有复杂性保证

它们既不分配内存，也不具有高于O(n log n)的时间复杂度

Note the exceptions of `stable_sort()`, `inplace_merge()`, and `stable_partition()`. Many implementations tend to temporarily allocate memory during these operations

#### 算法的性能与C库函数的等价一样好

标准 C 库配备了许多低级算法，包括`memcpy()`、`memmove()`、`memcmp()`和`memset()`。有时人们使用这些函数而不是标准算法库中的等价物。原因是人们倾向于相信 C 库函数更快，因此接受类型安全的折衷。

这对于现代标准库实现来说是不正确的；等价算法`std::copy()`、`std::equal()`和`std::fill()`在可能的情况下会使用这些低级 C 函数；因此，它们既提供性能又提供类型安全。

当然，也许会有例外情况，C++编译器无法检测到可以安全地使用低级 C 函数的情况。例如，如果一个类型不是平凡可复制的，std::copy()就不能使用memcpy()。但这是有充分理由的；希望一个不是平凡可复制的类的作者有充分的理由以这种方式设计类，我们（或编译器）不应该忽视这一点，而不调用适当的构造函数。

有时，C++算法库中的函数甚至比它们的 C 库等效函数表现得更好。最突出的例子是`std::sort()`与 C 库中的`qsort()`。`std::sort()`和`qsort()`之间的一个重大区别是，`qsort()`是一个函数，而`std::sort()`是一个函数模板。当`qsort()`调用比较函数时，由于它是作为函数指针提供的，通常比使用`std::sort()`时调用的普通比较函数慢得多，后者可能会被编译器内联

### 最佳实践

1. 使用受限算法：在 C++20 中引入的std::ranges下的受限算法比std下的基于迭代器的算法提供了一些优势。受限算法执行以下操作：

   * 支持投影，简化元素的自定义比较。
   * 支持范围而不是迭代器对。无需将begin()和end()迭代器作为单独的参数传递。
   * 易于正确使用，并且由于受 C++概念的限制，在编译期间提供描述性错误消息。

2. 仅对需要检索的数据进行排序，合理选择`sort()` 、 `partial_sort()` 和 `nth_element()`
   
3. 使用标准算法而不是原始的for循环

   * 标准算法提供了性能。即使标准库中的一些算法看起来很琐碎，它们通常以不明显的方式进行了最优设计。
   * 标准算法提供了安全性。即使是更简单的算法也可能有一些特殊情况，很容易忽视。
   * 标准算法是未来的保障；如果您想利用 SIMD 扩展、并行性甚至是以后的 GPU，可以用更合适的算法替换给定的算法（参见第十四章，并行算法）。
   * 标准算法有详细的文档。

## 6. 范围和视图

随着 C++20 引入 Ranges 库，我们在实现算法时从标准库中受益的方式得到了重大改进：

* 概念（Concepts）：定义了对迭代器和范围的要求，现在可以由编译器更好地检查，并在开发过程中提供更多的帮助。
* <algorithm> 头文件中所有函数的新重载版本都使用了上述概念进行约束，并接受范围作为参数，而不是迭代器对。
* 迭代器头文件中受约束的迭代器。
* 范围视图（Range views），使得算法可以组合。

算法库的局限之一体现在可组合性。如果我们有个Student类，需要得到特定年级的考试最高分：

```cpp
struct Student {
  int year_{};
  int score_{};
  std::string name_{};
  // ...
}; 

// 一般思路
auto get_max_score(const std::vector<Student>& students, int year) {
  auto by_year = = { return s.year_ == year; }; 
  // The student list needs to be copied in
  // order to filter on the year
  auto v = std::vector<Student>{};
  // 使用copy_if()和std::back_inserter()时会创建不必要的Student对象的副本
  std::ranges::copy_if(students, std::back_inserter(v), by_year); 
  auto it = std::ranges::max_element(v, std::less{}, &Student::score_);
  return it != v.end() ? it->score_ : 0; 
} 

// C20写法
auto max_value(auto&& range) {
  const auto it = std::ranges::max_element(range);
  return it != range.end() ? *it : 0;
}
auto get_max_score(const std::vector<Student>& students, int year) {
  const auto by_year = = { return s.year_ == year; };
  return max_value(students 
    | std::views::filter(by_year)
    | std::views::transform(&Student::score_));
} 
```

### 从Ranges库理解视图

**Views in the Ranges library are lazy evaluated iterations over a range. Technically, they are only iterators with built-in logic, but syntactically, they provide a very pleasant syntax for many common operations.**

```cpp
auto numbers = std::vector{1, 2, 3, 4};
auto square = [](auto v) {  return v * v; };  // 不是numbers值平方的副本
auto squared_view = std::views::transform(numbers, square);
for (auto s : squared_view) { // The square lambda is invoked here
  std::cout << s << " ";  // 每次访问时调用 std::transform()，即 lazy evaluated
}
```

如果要持久化，可以使用`std::ranges::copy()`将视图物化为容器。

```cpp
// 过滤视图，满足条件的元素，在遍历视图时可见
auto v = std::vector{4, 5, 6, 7, 6, 5, 4};
auto odd_view = 
  std::views::filter(v, [](auto i){ return (i % 2) == 1; });
for (auto odd_number : odd_view) {
  std::cout << odd_number << " ";
}
// Output: 5 7 5
```

```cpp
// 视图可以建立在多个可迭代容器上，看起来像是单一列表
auto list_of_lists = std::vector<std::vector<int>> {
  {1, 2},
  {3, 4, 5},
  {5},
  {4, 3, 2, 1}
};
auto flattened_view = std::views::join(list_of_lists);
for (auto v : flattened_view) 
  std::cout << v << " ";
// Output: 1 2 3 4 5 5 4 3 2 1

auto max_value = *std::ranges::max_element(flattened_view);
// max_value is 5 
```

#### 视图是可组合的

The full power of views comes from the ability to combine them. 

视图并不拷贝数据，因此可以在一个数据集合上表达多个操作，实际仅在内部迭代一次

```cpp
auto get_max_score(const std::vector<Student>& s, int year) {
  auto by_year = = { return s.year_ == year; };

  auto v1 = std::ranges::ref_view{s}; // Wrap container in a view
  auto v2 = std::ranges::filter_view{v1, by_year};
  auto v3 = std::ranges::transform_view{v2, &Student::score_};
  auto it = std::ranges::max_element(v3);
  return it != v3.end() ? *it : 0;
}

// 简写
using namespace std::ranges;
auto scores = transform_view{filter_view{ref_view{s}, by_year}, &Student::score_};

// 使用范围适配器
using namespace std::views;
auto scores = transform(filter(s, by_year), &Student::score_); 
```

总之，Ranges 库中的每个视图包括：

* 一个类模板（实际视图类型），它操作视图对象，例如`std::ranges::transform_view`。这些视图类型可以在命名空间`std::ranges`下找到。

* 一个范围适配器对象，它从范围创建视图类的实例，例如`std::views::transform`。所有范围适配器都实现了`operator()()`和`operator|()`，这使得可以使用管道运算符或嵌套来组合转换。范围适配器对象位于命名空间`std::views`下。

#### 视图是具有复杂性保证的非拥有范围

Ranges 库中的视图类型保证在常数时间复杂度内完成构造、复制和析构。

#### 视图不会改变底层容器

#### 视图可以实体化为容器

#### 视图是延迟计算的（惰性评估）

### 标准库的视图

在C++20之前，标准库也提供了类似的非拥有（Non-owning）视图, std::string_view std::span

#### Range views

```cpp
// 1. 生成
for (auto i : std::views::iota(-2, 2)) {
  std::cout << i << ' ';
}
// Prints -2 -1 0 1 

// 2. 转换
std::views::transform   // 转换元素
std::views::reverse     // 反转元素
std::views::split       // 分割元素
std::views::join        // 合并元素

auto csv = std::string{"10,11,12"};
auto digits = csv 
  | std::views::split(',')      // [ [1, 0], [1, 1], [1, 2] ]
  | std::views::join;           // [ 1, 0, 1, 1, 1, 2 ]
for (auto i : digits) {   std::cout << i; }
// Prints 101112 

// 3. 采样
std::views::filter     // 过滤元素
std::views::take       // 取前n个元素
std::views::drop       // 跳过前n个元素

auto vec = std::vector{1, 2, 3, 4, 5, 4, 3, 2, 1};
 auto v = vec
   | std::views::drop_while([](auto i) { return i < 5; })
   | std::views::take(3);
 for (auto i : v) { std::cout << i << " "; }
// Prints 5 4 3 

// 4. 实用
// 当您有想要转换或视为视图的东西时，它们非常方便。在这些视图类别中的一些示例是ref_view、all_view、subrange、counted和istream_view。
```

#### 重新审视 std::string_view 和 std::span

## 7.内存管理

大多数操作系统都是虚拟内存操作系统，它们提供了一个假象，即一个进程拥有了所有的内存。每个进程都有自己的**虚拟地址空间**。虚拟地址空间中的地址由操作系统和处理器的内存管理单元（MMU）映射到物理地址。每次访问内存地址时都会发生这种映射或转换。

分页&缺页中断

**进程内存**：

**栈**在许多方面与堆不同。以下是栈的一些独特属性：

* 栈是一个连续的内存块。
* 它有一个固定的最大大小。如果程序超出最大栈大小，程序将崩溃。这种情况称为栈溢出。
* 栈内存永远不会变得分散。
* 从栈中分配内存（几乎）总是很快的。页面错误可能会发生，但很少见。
* 程序中的每个线程都有自己的栈。

**堆**（*自由存储区*）

```cpp
{
  // C++17之前 placement_new
  auto* memory = std::malloc(sizeof(User));
  // ::前面的双冒号确保了从全局命名空间进行解析，以避免选择operator new的重载版本。
  auto* user = ::new (memory) User("john"); 

  user->~User();
  std::free(memory);
}

{
  // C++17, memory中引入了一些函数来实现
  // 使用一些以std::uninitialized_开头的函数来构造、复制和移动对象到未初始化的内存区域
  // 使用std::destroy_at()在特定内存地址上销毁对象，而无需释放内存
  auto* memory = std::malloc(sizeof(User));
  auto* user_ptr = reinterpret_cast<User*>(memory);
  std::uninitialized_fill_n(user_ptr, 1, User{"john"});
  std::destroy_at(user_ptr);
  std::free(memory); 
}

{
  // C++ 20 
  std::construct_at(user_ptr, User{"john"});        // C++20 
}
```

### 内存对齐

CPU 每次从内存中读取一个字时，将其读入寄存器。64 位架构上的字大小为 64 位，32 位架构上为 32 位。**对齐是一个实现定义的整数值，表示给定对象可以分配的连续地址之间的字节数。**

使用 `alignof` 操作符，如 `std::cout << alignof(int) << '\n';` 可查看 `int` 类型对齐要求，可能输出 4，即 4 字节对齐。

**可移植性问题**：C++ 标准未规定有效地址起始值，实际平台多从 0 开始，虽可用取模运算符检查，但编写完全可移植代码需用 `std::align()`

```cpp
bool is_aligned(void* ptr, std::size_t alignment) {
  assert(ptr != nullptr);
  // 确保 `ptr` 不为空，`alignment` 是 2 的幂（用 `std::has_single_bit()` 检查）
  assert(std::has_single_bit(alignment)); // Power of 2
  auto s = std::numeric_limits<std::size_t>::max();
  auto aligned_ptr = ptr;
  // 调用 `std::align()` 调整指针，比较原始指针和调整后指针判断是否已对齐
  std::align(alignment, 1, aligned_ptr, s);
  return ptr == aligned_ptr;
} 
```

`new` 和 `malloc()` 保证返回适合任何标量类型的内存，`<cstddef>` 中的 `std::max_align_t` 类型，对齐要求至少与所有标量类型一样严格，即使请求 `char` 内存，也适合 `std::max_align_t`。

**连续分配 `char` 情况**：连续用 `new` 分配 `char`，`p1` 和 `p2` 间空间取决于 `std::max_align_t` 对齐要求，如系统中为 16 字节，每个 `char` 实例间有 15 字节。

**声明变量时指定**：使用 `alignas` 指定符，如 `alignas(64) int x{}; alignas(64) int y{};` 可确保 `x` 和 `y` 位于不同缓存行。

**定义类型时指定**：如 `struct alignas(64) CacheLine { std::byte data[64]; };` 创建的结构体对象会按 64 字节自定义对齐。

**堆上分配与非默认对齐** C++17 引入 `operator new()` 和 `operator delete()` 新重载，接受 `std::align_val_t` 类型对齐参数。`<cstdlib>` 中 `aligned_alloc()` 函数可手动分配对齐的堆内存。

```cpp
constexpr auto ps = std::size_t{4096};      // Page size
struct alignas(ps) Page {
    std::byte data_[ps];
};
auto* page = new Page{};                    // Memory page
assert(is_aligned(page, ps));               // True
// Use page ...
delete page; 
```

**注意事项**

- 内存页面不是 C++ 抽象机器一部分，无可移植方法编程获取当前系统页面大小，Unix 系统可用 `boost::mapped_region::get_page_size()` 或 `getpagesize()`。
- 支持的对齐集由使用的标准库实现定义，而非 C++ 标准。

**填充**，注意设置数据成员的顺序，可以起到节约内存的效果：

```cpp
class Document {
  bool is_cached_{};
  std::byte padding1[7]; // Invisible padding inserted by compiler
  double rank_{};
  int id_{};
  std::byte padding2[4]; // Invisible padding inserted by compiler
}; 

std::cout << sizeof(Document) << '\n'; // Possible output is 24 

// Version 2 of Document class after padding
class Document { 
  double rank_{}; 
  int id_{}; 
  bool is_cached_{}; 
  std::byte padding[3]; // Invisible padding inserted by compiler 
};

std::cout << sizeof(Document) << '\n'; // Possible output is 16 
```

### 内存所有权

* 局部变量由当前作用域拥有。当作用域结束时，在作用域内创建的对象将被自动销毁
* 静态和全局变量由程序拥有，并将在程序终止时被销毁
* 数据成员由它们所属的类的实例拥有
* 只有动态变量没有默认所有者，程序员需要确保所有动态分配的变量都有一个所有者来控制变量的生命周期

明确表达所有权，最小化手动内存管理

#### 隐式处理资源

通过RAII技术隐式处理资源

可以使用标准容器来处理对象的集合（通过容器管理动态内存，而不是手动 `new` 和 `delete`）

使用 `std::optional` 来处理可能存在或可能不存在的对象的生命周期

尽量使用智能指针代替裸指针：

* 独占指针也非常高效，因为与普通原始指针相比，它们几乎没有性能开销。轻微的开销是由于`std::unique_ptr`具有非平凡的析构函数，这意味着（与原始指针不同）在传递给函数时无法将其传递到 CPU 寄存器中。这使它们比原始指针慢。

### 小对象优化

```cpp
// 字符串的小对象优化的简单示意
struct Long { 
  size_t capacity_{}; 
  size_t size_{}; 
  char* data_{}; 
}; 

struct Short { 
  unsigned char size_{};
  char data_[23]{}; 
};

union u_ { 
  Short short_layout_; 
  Long long_layout_; 
};
```

libc++在长模式下使用capacity_数据成员的最低有效位，而在短模式下使用size_数据成员的最低有效位。对于长模式，这个位是多余的，因为字符串总是分配 2 的倍数的内存大小。在短模式下，可以只使用 7 位来存储大小，以便一个位可以用于标志。

### 自定义内存管理

> 可以结合 内存分配算法 + PMR 实现一个例子

## 8. 编译时编程

### 模版元编程

略过部分基础内容

C++20 引入了一种新的缩写语法，用于编写函数模板，采用了通用 lambda 使用的相同风格。通过使用auto作为函数参数类型，我们实际上创建的是一个函数模板，而不是一个常规函数

```cpp
// pow_n accepts any number type 
template <typename T> 
auto pow_n(const T& v, int n) { 
  auto product = T{1}; 
  for (int i = 0; i < n; ++i) { 
    product *= v; 
  }
  return product; 
} 

// C20
auto pow_n(const auto &v, int n) {  // Declares a function template
  typename std::remove_cvref<decltype(v)>::type product{1}; 
  for (int i = 0; i < n; ++i) { product *= v; } 
  return product;
}
```

在 C++20 之前，在通用 lambda 的主体中经常看到decltype。然而，现在可以通过向通用 lambda 添加显式模板参数来避免相当不方便的decltype:

```cpp
auto pow_n = []<class T>(const T& v, int n) { 
  auto product = T{1};
  for (int i = 0; i < n; ++i) { product *= v; }
  return product;
}; 
```

### 类型特征（`<type_traits>`）

为了提取有关模板类型的信息，标准库提供了一个类型特征库，该库在`<type_traits>`头文件中可用。所有类型特征都在编译时评估。

有两类类型特征：

* 返回关于类型信息的类型特征，作为布尔值或整数值。
* 返回新类型的类型特征。这些类型特征也被称为元函数。

### 常量表达式

`constexpr`关键字也可以与函数一起使用。在这种情况下，它告诉编译器某个函数打算在编译时评估。

`constexpr`函数有一些限制；不允许执行以下操作：

* 处理本地静态变量
* 处理thread_local变量
* 调用任何函数，本身不是constexpr函数

```cpp
constexpr auto sum(int x, int y, int z) { return x + y + z; } 

constexpr auto value = sum(3, 4, 5); 
// 由于sum()的结果用于常量表达式，并且其所有参数都可以在编译时确定，因此编译器将生成
const auto value = 12; 
// 调用sum()并将结果存储在未标记为constexpr的变量中，编译器可能（很可能）在编译时评估sum()
auto value = sum(3, 4, 5); // value is not constexpr 
```

`constexpr`函数可以在运行时或编译时调用。如果我们想限制函数的使用，使其只在编译时调用，我们可以使用关键字`consteval`而不是`constexpr`。假设我们想禁止在运行时使用`sum()`。使用 C++20，我们可以通过以下代码实现：

```cpp
// 立即函数
consteval auto sum(int x, int y, int z) { return x + y + z; } 
```

`if constexpr`编译期条件判断语句，它允许在编译期间根据条件选择是否编译某段代码。

```cpp
if constexpr (constant_expression) {
    // 如果 constant_expression 为 true，则编译这段代码，忽略else分支
} else {
    // 如果 constant_expression 为 false，则编译这段代码，忽略if分支
}
```

下面例子会编译失败，因为编译器不理解程序分支是否可达，在代码生成时会生成对应的else分支。

```cpp
template <typename T> 
auto generic_mod(const T& v, const T& n) -> T {
  assert(n != 0);
  if (std::is_floating_point_v<T>) { return std::fmod(v, n); }
  else { return v % n; }
} 

// 等价于
auto generic_mod(const float& v, const float& n) -> float {
  assert(n != 0);
  if (true) { return std::fmod(v, n); }
  else { return v % n; } // Will not compile
} 

```

改写为 `if constexpr` 可以通过编译：

```cpp
template <typename T> 
auto generic_mod(const T& v, const T& n) -> T { 
  assert(n != 0);
  if constexpr (std::is_floating_point_v<T>) {
    return std::fmod(v, n);
  } else {                 // If T is a floating point,
    return v % n;          // this code is eradicated
  }
} 

// 等价于
auto generic_mod(const float& v, const float& n) -> float { 
  assert(n != 0);
  return std::fmod(v, n); 
} 

```

使用`static_assert()`在编译程序时捕获编程错误

### `concept`和`requires`

![error in template compile](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CppLearn/error_in_template_compile.jpg
)

```cpp
// 基本例子
template <typename T>
concept FloatingPoint = std::is_floating_point_v<T>; 

// 可以使用逻辑运算符组合多个约束条件
template <typename T>
concept Number = FloatingPoint<T> || std::is_integral_v<T>;

// 可以使用 requires 来添加一组语句到 concept 中
template<typename T>
concept range = requires(T& t) {
  ranges::begin(t);
  ranges::end(t);
};
```

使用概念约束类型

```cpp
// 使用 concept 约束模版函数 或 模板类
template <typename T>
requires std::integral<T>
auto mod(T v, T n) { 
  return v % n;
}

template <typename T>
requires std::integral<T>
struct Foo {
  T value;
};

// 更紧凑的写法
template <std::integral T>
auto mod(T v, T n) { 
  return v % n;
}

template <std::integral T>
struct Foo {
  T value;
};

// 定义函数模板时使用缩写的函数模板形式，可以在auto关键字前面添加 concept
auto mod(std::integral auto v, std::integral auto n) {
  return v % n;
}

// 返回类型也可以通过使用概念来约束
std::integral auto mod(std::integral auto v, std::integral auto n) {
  return v % n;
} 
```

Point2D模板的约束版本

```cpp
template<typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template<typename T>
concept Point = requires<T p> {
  requires std::is_same_v<decltype(p.x()), decltype(p.y())>;
  requires Arithmetic<decltype(p.x())>;
};

std::floating_point auto dist(Point auto p1, Point auto p2) {
  ……
}

template <Arithmetic T> // T is now constrained!
class Point2D {
public:
  Point2D(T x, T y) : x_{x}, y_{y} {}
  auto x() { return x_; }
  auto y() { return y_; }
  // ...
private:
  T x_{};
  T y_{};
}; 
```

接口现在更加描述性，当我们尝试用错误的参数类型实例化它时，我们将在实例化阶段而不是最终编译阶段获得错误。

类型和概念都指定了对象上支持的一组操作。通过检查类型或概念，我们可以确定某些对象如何构造、移动、比较和通过成员函数访问等。

一个重大的区别是，概念并不说任何关于对象如何存储在内存中，而类型除了其支持的操作集之外还提供了这些信息。例如，我们可以在类型上使用sizeof运算符，但不能在概念上使用。

C++20 还包括一个新的 `<concepts>` 头文件，其中包含预定义的概念。您已经看到其中一些概念的作用。许多概念都是基于类型特性库中的特性。然而，有一些基本概念以前没有用特性表达。其中最重要的是比较概念，如 `std::equality_comparable` 和 `std::totally_ordered`，以及对象概念，如 `std::movable`、`std::copyable`、`std::regular` 和 `std::semiregular`。我们不会在标准库的概念上花费更多时间，但在开始定义自己的概念之前，请记住将它们牢记在心。在正确的泛化级别上定义概念并不是件容易的事，通常明智的做法是基于已经存在的概念定义新的概念。


## 9.基本实用程序

介绍 C++实用库中的一些基本类

* 使用std::optional表示可选值

* 使用std::pair、std::tuple和std::tie()来固定大小的集合

* 使用标准容器存储具有std::any和std::variant类型的元素的动态大小集合

### `std::optional<T>`

`std::optional`的语法类似于指针；值通过`operator*()`或`operator->()`访问。尝试使用`operator*()`或`operator->()`访问空的可选值的值是未定义行为。还可以使用`value()`成员函数访问值，如果可选值不包含值，则会抛出`std::bad_optional_access`异常。

如果对std::optional<T>的容器进行排序，空的可选值将出现在容器的开头，而非空的可选值将像通常一样排序

### 固定大小异构集合

`std::pair` `std::tuple`

C++11 引入了一个名为std::tuple的新实用类，它是std::pair的泛化，可以容纳任意数量的元素。

```cpp
auto t = std::tuple<int, std::string, bool>{}; 
```

可以使用自由函数模板`std::get<Index>()`访问std::tuple的各个元素。std::tuple类基本上是一个简单的结构，其成员可以通过编译时索引访问。

std::tuple包含不同类型的元素，而基于范围的 for 循环中const auto& v的类型只会被评估一次，无法适配元组中多种不同类型的元素，因此这类代码无法编译。

同时，由于迭代器不能改变指向的类型，std::tuple不提供begin()、end()成员函数及下标运算符[]，常规迭代算法也不适用，所以需要其他方法来展开元组。

[godbolt](https://godbolt.org/z/nj9s7xjYf)

```cpp
// 为特定索引生成调用的函数
template <size_t Index, typename Tuple, typename Func> 
void tuple_at(const Tuple& t, Func f) {
  const auto& v = std::get<Index>(t);
  std::invoke(f, v);
} 

// 如果索引等于元组大小，它会生成一个空函数。否则，它会在传递的索引处执行 lambda，并生成一个索引增加 1 的新函数
template <typename Tuple, typename Func, size_t Index = 0> 
void tuple_for_each(const Tuple& t, const Func& f) {
  constexpr auto n = std::tuple_size_v<Tuple>;
  if constexpr(Index < n) {
    tuple_at<Index>(t, f);
    tuple_for_each<Tuple, Func, Index+1>(t, f);
  }
}

auto t = std::tuple{1, true, std::string{"Jedi"}};
tuple_for_each(t, [](const auto& v) { std::cout << v << " "; });
// Prints "1 true Jedi" 
```

在tuple_for_each()的基础上，可以以类似的方式实现迭代元组的不同算法。以下是std::any_of()为元组实现的示例

```cpp
template <typename Tuple, typename Func, size_t Index = 0> 
auto tuple_any_of(const Tuple& t, const Func& f) -> bool { 
  constexpr auto n = std::tuple_size_v<Tuple>; 
  if constexpr(Index < n) { 
    bool success = std::invoke(f, std::get<Index>(t)); 
    if (success) {
      return true;
    }
    return tuple_any_of<Tuple, Func, Index+1>(t, f); 
  } else { 
    return false; 
  } 
} 
```

C++17 引入结构化绑定来优雅地访问元组中的元素

```cpp
auto make_saturn() { return std::tuple{"Saturn"s, 82, true}; }
const auto& [name, n_moons, rings] = make_saturn();
std::cout << name << ' ' << n_moons << ' ' << rings << '\n'; 
```

**可变模板参数包使程序员能够创建可以接受任意数量参数的模板函数。**

参数包通过在类型名称前面放置三个点和在可变参数后面放置三个点来识别，用逗号分隔扩展包：

```cpp
template<typename ...Ts> 
auto f(Ts... values) {
  g(values...);
}
```

### 动态大小的异构集合

`std::any` `std::variant`

```cpp
// 使用std::any作为基本类型。
// 缺点是性能不好：
//  每次访问其中的值时，必须在运行时测试类型，编译时完全失去了存储值的类型信息；
//  在堆上分配对象而不是栈上
auto container = std::vector<std::any>(42, "hi", true);
```

[`std::variant`](https://en.cppreference.com/w/cpp/utility/variant.html)：The class template std::variant represents a type-safe union.

* 它不会将其包含的类型存储在堆上（不像std::any）
* 它可以通过通用 lambda 调用，这意味着您不必明确知道其当前包含的类型

```cpp
// 用 std::holds_alternative<T>() 来检查变体当前是否持有给定类型
using VariantType = std::variant<int, std::string, bool>; 
VariantType v{}; 
std::holds_alternative<int>(v);  // true, int is first alternative
v = 7; 
std::holds_alternative<int>(v);  // true
v = std::string{"Anne"};
std::holds_alternative<int>(v);  // false, int was overwritten 
v = false; 
std::holds_alternative<bool>(v); // true, v is now bool 
```

简单来说，`std::variant<T1, T2, ..., Tn>` 的大小遵循：`sizeof(std::variant) = 最大sizeof(Ti) + 判别器大小 + 可能的对齐填充`

#### std::variant 的异常安全性

`std::variant` 的构造过程可能因存储类型的构造函数抛出异常而失败。此时：

- 如果在初始化第一个（或唯一）可能的类型时抛出异常，`variant` 会进入 **`valueless_by_exception` 状态**（不持有任何有效值）。
- 这种状态是安全的：`variant` 对象本身仍可正常析构，不会导致资源泄漏，也不会产生未定义行为。
- 例：`std::variant<int, std::string> v("hello");` 若 `std::string` 的构造抛出异常（如内存分配失败），`v` 会进入 `valueless_by_exception` 状态。

对 `std::variant` 进行赋值（`operator=`）或修改（如 `emplace`、`swap`）时，异常安全取决于具体操作：

- **`emplace` 操作**：  
  直接在 `variant` 内部构造新值以替换当前值。若新值的构造函数抛出异常：
  - 若替换前 `variant` 已持有值，旧值会被销毁，`variant` 进入 `valueless_by_exception` 状态（基本异常安全：对象状态有效但可能丢失原有值）。
  - 若 `variant` 原本就是 `valueless_by_exception` 状态，异常后仍保持该状态。

- **赋值操作（`operator=`）**：  
  当赋予新类型或同类型值时，实现通常采用“复制-交换”策略：
  1. 先构造一个临时 `variant` 持有新值；
  2. 若构造成功，与当前 `variant` 交换状态；
  3. 若临时 `variant` 构造失败，当前 `variant` 状态不变（**强异常安全保证**）。

- **`swap` 操作**：  
  交换两个 `variant` 的状态，通常是 noexcept 的（若存储类型的交换操作 noexcept）。若交换过程中抛出异常（极少发生），标准未强制规定状态，但实践中会保证两个 `variant` 处于有效状态（可能交换不完整，但无资源泄漏）。

`valueless_by_exception` 是 `std::variant` 专门设计的“安全无效状态”，用于应对构造/修改时的异常：

- 可通过 `valueless_by_exception()` 方法检测该状态。
- 处于该状态的 `variant` 仍可正常析构，也可被重新赋值为有效状态。
- 若对该状态的 `variant` 调用 `get<T>()` 或 `visit`，会抛出 `std::bad_variant_access` 异常（属于预期行为，而非安全问题）。


`std::variant` 的析构函数始终安全：

- 若持有有效值，会调用对应类型的析构函数释放资源。
- 若处于 `valueless_by_exception` 状态，析构函数无操作（无需释放资源）。
- 无论哪种情况，都不会导致资源泄漏。

使用时需注意检测 `valueless_by_exception` 状态，避免对无效状态的 `variant` 进行访问操作。

#### 访问变体

访问std::variant中的变量时，我们使用全局函数std::visit()，通常通过 lambda 来访问

```cpp
auto var = std::variant<int, bool, float>{};
std::visit([](auto&& val) { std::cout << val; }, var); 

// 使用通用 lambda 和变体var调用std::visit()时，编译器会将 lambda 概念上转换为一个常规类
// 该类对变体中的每种类型进行operator()重载，类似如下：
struct GeneratedFunctorImpl {
  auto operator()(int&& v)   { std::cout << v; }
  auto operator()(bool&& v)  { std::cout << v; }
  auto operator()(float&& v) { std::cout << v; }
}; 

// std::visit()函数扩展为使用std::holds_alternative<T>()的if...else链
// 或使用std::variant的索引生成正确的调用std::get<T>()的跳转表。
```

#### 全局函数 `std::get()`

全局函数模板`std::get()`可用于`std::tuple`、`std::pair`、`std::variant`和`std::array`。有两种实例化std::get()的方式，一种是使用索引，一种是使用类型：

* `std::get<Index>()`: 当`std::get()`与索引一起使用时，如`std::get<1>(v)`，它返回std::tuple、std::pair或std::array中相应索引处的值。

* `std::get<Type>()`: 当`std::get()`与类型一起使用时，如`std::get<int>(v)`，返回std::tuple、std::pair或std::variant中的相应值。对于std::variant，如果变体当前不持有该类型，则会抛出std::bad_variant_access异常。请注意，如果v是std::tuple，并且Type包含多次，则必须使用索引来访问该类型。

## 10. 代理对象和延迟评估

> Proxy Objects and Lazy Evaluation

First and foremost, the techniques used in this chapter are used to hide optimizations in a library from the user of that library. This is useful because exposing every single optimization technique as a separate function requires a lot of attention and education from the user of the library. It also bloats the code base with a multitude of specific functions, making it hard to read and understand. By using proxy objects, we can achieve optimizations under the hood; the resultant code is both optimized and readable.

**Lazy evaluation** is a technique used to postpone an operation until its result is really needed. The opposite, where operations are performed right away, is called **eager evaluation**.

* 延迟执行：所有计算仅在必要时触发（如比较、赋值），避免提前消耗资源。
* 零额外开销：代理对象仅存储引用或指针，不复制原始数据，符合 “零成本抽象” 原则。
* 语法透明性：通过运算符重载使代理对象的使用方式与原始对象一致，不破坏代码可读性。

一个简单的例子：

```cpp
class Image { /* ... */ };                   // Buffer with JPG data
auto load(std::string_view path) -> Image;   // Load image at path
class ScoreView {
public:
  // Eager, requires loaded bonus image
  void display(const Image& bonus);
  // Lazy, only load bonus image if necessary
  void display(std::function<Image()> bonus);
  // ...
}; 

// Always load bonus image eagerly
const auto eager = load("/images/stars.jpg");
score.display(eager); 

// Load default image lazily if needed
auto lazy = [] { return load("/images/stars.jpg"); }; 
score.display(lazy); 
```

A technique for hiding the fact that the code evaluates lazily is to use **proxy objects**.

### 使用代理对象的一些例子

1. 通过代理对象延迟字符串拼接操作，避免创建临时对象以提升性能。

```cpp
class StringProxy {
private:
    const std::string& left_;
    const std::string& right_;
public:
    StringProxy(const std::string& l, const std::string& r) : left_(l), right_(r) {}

    // 延迟拼接，仅在需要时执行
    operator std::string() const {
        return left_ + right_;
    }

    // 重载比较运算符，直接操作原始字符串避免拼接
    bool operator==(const std::string& other) const {
        if (left_.size() + right_.size() != other.size()) return false;
        return other.substr(0, left_.size()) == left_ &&
               other.substr(left_.size()) == right_;
    }
};

// 使用示例
std::string a = "Hello", b = "World";
StringProxy proxy(a, b);
if (proxy == "HelloWorld") {  // 不触发拼接，直接比较
    std::string s = proxy;    // 此时才执行拼接
}
```

2. 二维向量长度比较场景，通过LengthProxy延迟sqrt计算，仅在必要时执行

```cpp
class Vector2D {
private:
    float x_, y_;
public:
    Vector2D(float x, float y) : x_(x), y_(y) {}

    // 返回代理对象而非直接计算长度
    class LengthProxy {
    private:
        const Vector2D& vec_;
    public:
        LengthProxy(const Vector2D& v) : vec_(v) {}

        // 延迟计算：仅在比较时执行sqrt
        bool operator<(const LengthProxy& other) const {
            // 比较平方值以避免sqrt，进一步优化
            return (vec_.x_ * vec_.x_ + vec_.y_ * vec_.y_) <
                   (other.vec_.x_ * other.vec_.x_ + other.vec_.y_ * other.vec_.y_);
        }
    };

    LengthProxy length() const { return LengthProxy(*this); }
};

// 使用示例
Vector2D v1(3, 4), v2(1, 2);
if (v1.length() < v2.length()) {  // 不计算sqrt，直接比较平方和
    // ...
}
```

3. 利用代理对象和运算符重载实现类似 “扩展方法” 的链式调用

```cpp
// 代理类：包装值并支持管道操作
template <typename T>
class PipeProxy {
private:
    T value_;
public:
    PipeProxy(T val) : value_(std::move(val)) {}

    // 重载管道运算符，接收函数并返回新代理
    template <typename F>
    auto operator|(F&& func) const {
        return PipeProxy<std::invoke_result_t<F, T>>(func(value_));
    }

    // 提取最终值
    const T& get() const { return value_; }
};

// 示例函数：字符串处理
std::string to_upper(const std::string& s) {
    std::string res = s;
    std::transform(res.begin(), res.end(), res.begin(), ::toupper);
    return res;
}

std::string trim(const std::string& s) {
    auto start = s.begin();
    while (start != s.end() && std::isspace(*start)) start++;
    auto end = s.end();
    while (end != start && std::isspace(*(end-1))) end--;
    return std::string(start, end);
}

// 使用示例
std::string s = "  hello world  ";
auto result = PipeProxy(s) | trim | to_upper;  // 链式调用
std::cout << result.get();  // 输出 "HELLO WORLD"
```

## 11. 并发

### 并发和并行

并发（Concurrency）

* **任务执行方式** ：多个任务同时处于执行状态，但它们可能并不是同时在物理上执行。在单核处理器系统中，操作系统通过快速切换任务（时间片轮转）来让多个务交替执行，给人以同时执行的错觉。
* **关注重点** ：在于让多个任务同时进行，充分利用系统资源，提高整体效率，强调的是任务的执行方式和调度机制。
* **实现方式** ：可以通过多线程、进程间通信、协程等方式来实现，不依赖硬件的多核支持。
* **性能提升** ：在单核上并发执行任务不一定能提高执行速度，甚至可能因为任务切换开销而变慢，但在多核环境下可以提升效率。
* **适用场景** ：适用于 I/O 密集型任务，如网络请求、文件读写等，任务等待时间长，通过并发可以提高资源利用率。

并行（Parallelism）

* **任务执行方式** ：多个任务同时在多个处理器或核心上同时执行，真正利用了多核硬件资源来加速任务执行。
* **关注重点** ：在于任务的分解和同时执行，以减少完成任务所需的时间，强调的是任务的实际同时执行和计算能力的提升。
* **实现方式** ：依赖于多核处理器或多个计算节点，在每个处理器或核心上分配子任务同时执行。
* **性能提升** ：能充分利用多核硬件资源，显著提高任务执行速度，理论上性能提升与处理器核心数成正比。
* **适用场景** ：适用于计算密集型任务，如科学计算、大数据处理等，需要大量计算资源，通过并行可以快速得到结果。

#### 共享内存

#### 数据竞争

避免数据竞争：

* 使用原子数据类型而不是int。这将告诉编译器以原子方式执行读取、增加和写入。我们将在本章后面花更多时间讨论原子数据类型。

* 使用互斥锁（mutex）来保证多个线程永远不会同时执行关键部分。关键部分是代码中不得同时执行的地方，因为它更新或读取可能会产生数据竞争的共享内存。

#### 互斥锁和临界区

互斥锁，是用于避免数据竞争的同步原语。需要进入临界区的线程首先需要锁定互斥锁

#### 死锁

使用互斥锁保护共享资源时，存在陷入死锁状态的风险。当两个线程互相等待对方释放锁时，就会发生死锁。

### 并发编程

```cpp
std::this_thread::get_id();  // 获取线程标识符

std::this_thread::sleep_for(std::chrono::seconds{1});  // 线程休眠

std::thread::hardware_concurrency(); // 硬件线程的总数
```

[**C++20 `jthread`**](https://en.cppreference.com/w/cpp/thread/jthread.html)

C++20 引入了 `std::jthread`，它是一种更智能的线程管理方式，解决了 `std::thread` 在生命周期管理和线程取消方面的不足。

`std::jthread` 对象在被销毁时会自动调用 `join()` 方法，确保在对象生命周期结束时，关联的线程会完成或终止。这避免了手动调用 `join` 或 `detach` 的麻烦，以及可能引发的资源泄漏问题。

```cpp
std::jthread jthr{[]{std::this_thread::sleep_for(std::chrono::seconds(1));}};
// jthr 自动 join，无需手动调用
```

`std::jthread` 与 `std::stop_source` 和 `std::stop_token` 紧密集成，允许外部请求线程停止。线程函数可以通过检查 `std::stop_token` 的状态来优雅地停止执行。

```cpp
auto stop_source = std::stop_source{};
std::jthread jthr{[&]{ 
    std::stop_token stok = stop_source.get_token();
    while (!stok.stop_requested()) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}}; 
stop_source.request_stop(); // 外部请求线程停止
```

**使用`std::mutex`保护关键部分**

```cpp
#include <mutex>
#include <thread>
#include <cassert>

std::mutex counter_mutex; // 用于保护counter的互斥锁
int counter = 0;

void increment_counter(int n) {
    for (int i = 0; i < n; ++i) {
        std::lock_guard<std::mutex> lock(counter_mutex); // 自动管理锁
        ++counter;
    }
}
```

**避免死锁，死锁是指两个或多个线程因为等待对方释放资源而无法继续执行的情况。为了避免死锁，可以使用std::lock()函数同时锁定多个互斥锁**

```cpp
#include <mutex>

struct Account {
    int balance_ = 0;
    std::mutex m_;
};

void transfer_money(Account& from, Account& to, int amount) {
    std::unique_lock<std::mutex> lock1(from.m_, std::defer_lock);
    std::unique_lock<std::mutex> lock2(to.m_, std::defer_lock);
    std::lock(lock1, lock2);

    from.balance_ -= amount;
    to.balance_ += amount;
}
```

**使用条件变量协调线程**

```cpp
#include <condition_variable>
#include <queue>
#include <thread>

std::condition_variable cv;
std::queue<int> q;
std::mutex mtx; // 保护共享队列
const int sentinel = -1;

void print_ints() {
    int i = 0;
    while (i != sentinel) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [] { return !q.empty(); });
        i = q.front();
        q.pop();
        if (i != sentinel) {
            std::cout << "Got: " << i << '\n';
        }
    }
}

void generate_ints() {
    for (int i : {1, 2, 3, sentinel}) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::lock_guard<std::mutex> lock(mtx);
        q.push(i);
        cv.notify_one();
    }
}

int main() {
    std::jthread producer(generate_ints);
    std::jthread consumer(print_ints);
}
```

**使用`std::future`和`std::promise`处理返回值和错误**

```cpp
#include <exception>
#include <future>
#include <iostream>

std::promise<int> p;

void divide(int a, int b) {
    if (b == 0) {
        std::runtime_error e("Divide by zero exception");
        p.set_exception(std::make_exception_ptr(e));
    } else {
        int result = a / b;
        p.set_value(result);
    }
}

int main() {
    std::thread(divide, 45, 5).detach();
    std::future<int> f = p.get_future();
    try {
        const int& result = f.get();
        std::cout << "Result: " << result << '\n';
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << '\n';
    }
}
```

`std::packaged_task`是一个可调用对象，它可以自动创建`std::promise`和`std::future`

```cpp
#include <future>
#include <iostream>

int divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("Divide by zero exception");
    }
    return a / b;
}

int main() {
    std::packaged_task<int(int, int)> task(divide);
    std::future<int> f = task.get_future();
    std::thread(std::move(task), 45, 5).detach();

    try {
        const int& result = f.get();
        std::cout << "Result: " << result << '\n';
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << '\n';
    }
}
```

`std::async`是一个函数模板，可以自动创建`std::packaged_task`和`std::thread`

```cpp
#include <future>
#include <iostream>

int divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("Divide by zero exception");
    }
    return a / b;
}

int main() {
    std::future<int> f = std::async(divide, 45, 5);
    try {
        const int& result = f.get();
        std::cout << "Result: " << result << '\n';
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << '\n';
    }
}
```

#### C++20中的同步原语

详见 [C++20 Additional synchronization primitives](../../Basic_Concept/C++_Advanced/20_05_additional_concurrency_support.md)

#### C++中的原子支持

对于所有标量数据类型，都有命名为std::atomic_int的 typedef。这与std::atomic<int>相同。

只要自定义类型是平凡可复制的，就可以将自定义类型包装在std::atomic模板中。基本上，这意味着类的对象完全由其数据成员的位描述。这样，对象可以通过例如std::memcpy()仅复制原始字节来复制。

原子变量可能会或可能不会使用锁来保护数据；这取决于变量的类型和平台。可以进行检查

```cpp
auto variable = std::atomic<int>{1};
assert(variable.is_lock_free());          // Runtime assert 

static_assert(std::atomic<int>::is_always_lock_free);   // since c++17, compile time check
```

在现代平台上，任何`std::atomic<T>`，其中T适合本机字大小，通常都是始终无锁的。在现代 x64 芯片上，甚至可以获得双倍的数量。例如，在现代英特尔 CPU 上编译的 libc++上，`std::atomic<std::complex<double>>`始终是无锁的。

#### 内存模型和无锁编程

话题较大，文档简单描述，需要参考其他材料

### 性能指南

* 避免竞争
* 避免阻塞操作
* 控制线程数：一个好方法是使用可以根据当前硬件大小调整大小的线程池
* 线程优先级：与线程优先级相关的一种可能会影响性能并且应该避免的现象称为优先级反转。当一个具有高优先级的线程正在等待获取当前由低优先级线程持有的锁时，就会发生这种情况。这种依赖关系会影响高优先级线程，因为它被阻塞，直到下一次低优先级线程被调度以释放锁。
* 线程亲和性：如果可能的话，一些线程应该在特定的核心上执行，以最小化缓存未命中

### 伪共享

[Thing_about_false_sharing](../../Basic_Concept/Things_About/Things_about_false_sharing.md)

## 12. 协程和惰性生成器（Coroutine & Lzay Generator）

### Abstraction

Subroutines and coroutines

> Coroutines are very similar to subroutines. In C++, we don't have anything explicitly called subroutines; instead, we write functions (free functions or member functions, for example) to create subroutines.

**To summarize, coroutines are subroutines that also can be suspended and resumed.**

* `Program counter (PC)`: The register that stores the memory address of the currently executing instruction. This value is automatically incremented whenever an instruction is executed. Sometimes it is also called an instruction pointer
* `Stack pointer (SP)`: It stores the address of the top of the currently used call stack. Allocating and deallocating stack memory is a matter of changing the value stored in this single register.

调用惯例：The exact details about how function calls are carried out are specified by something called calling conventions. They provide a protocol for the caller/callee to agree
on who is responsible for which parts. Calling conventions differ among CPU architectures and compilers and are one of the major parts that constitutes an application binary interface (ABI).

当调用函数时，该函数的调用帧（或激活帧）被创建。调用帧包含：

* 传递给函数的参数。
* 函数的局部变量。
* 打算使用的寄存器的快照，因此需要在返回之前恢复。
* 返回地址，它链接回调用者从中调用函数的内存位置。
* 可选的帧指针，指向调用者的调用帧顶部。在检查堆栈时，帧指针对调试器很有用。我们在本书中不会进一步讨论帧指针。

当子程序返回给调用者时，它使用返回地址来知道要跳转到哪里，恢复它已经改变的寄存器，并弹出（释放）整个调用帧。

### 有栈协程和无栈协程

根据是否拥有独立调用栈，协程可分为：

**有栈协程（Stackful Coroutine）**：每个协程拥有**独立的调用栈**（通常是固定大小或可动态增长的内存块），用于保存函数调用链、局部变量、返回地址等完整执行上下文。  

* 类比：可理解为“迷你线程”，但调度完全在用户态完成；  
* 核心特点：能在**任意函数调用位置**挂起（suspend）和恢复（resume），调度灵活性极高。

**无栈协程（Stackless Coroutine）**：协程没有独立的调用栈，其执行上下文（局部变量、程序计数器等）通过**状态机、闭包或编译器生成的结构体**保存，依赖语言或编译器的静态分析。 

* 类比：更像“状态驱动的函数”，执行流程被拆分为多个状态，挂起和恢复本质是状态的切换；  
* 核心特点：只能在**预定义的挂起点**（如`yield`、`await`关键字处）挂起，灵活性受限，但内存开销更小。

| 维度                | 有栈协程（Stackful）                          | 无栈协程（Stackless）                          |
|---------------------|----------------------------------------------|----------------------------------------------|
| **调用栈**          | 有独立调用栈（大小通常可配置）                 | 无独立调用栈，上下文保存在状态机/闭包中         |
| **挂起位置**        | 任意函数调用处（支持嵌套函数中挂起）           | 仅能在预定义挂起点（如`await`/`yield`处）挂起   |
| **内存开销**        | 较高（每个协程需分配栈内存，通常KB级）         | 极低（仅保存必要状态，通常几十到几百字节）       |
| **调度灵活性**      | 高（可在任意位置切换，支持复杂嵌套逻辑）       | 低（挂起点固定，依赖显式关键字标记）             |
| **实现方式**        | 多通过库实现（依赖汇编或操作系统API操作栈）     | 多依赖编译器支持（编译期将协程转换为状态机）     |
| **上下文切换成本**  | 较低（仅切换栈指针和寄存器，用户态操作）       | 极低（仅切换状态变量，几乎无开销）               |
| **适用场景**        | 复杂嵌套逻辑、通用并发（如游戏逻辑、多任务调度） | 简单异步流程（如IO密集型任务、网络请求）         |

不同语言根据设计目标（性能、易用性、兼容性）选择了不同的协程实现，以下是典型案例：

#### Python：`greenlet`库

* **特点**：Python标准库无原生有栈协程，`greenlet`是第三方库，通过C扩展实现有栈协程；  
* **原理**：每个`greenlet`对象拥有独立的栈，通过`switch()`方法在协程间切换，可在任意函数调用处挂起；  

```python
from greenlet import greenlet

def func1():
    print("进入func1")
    gr2.switch()  # 挂起func1，切换到gr2
    print("回到func1")  # 从gr2切换回来后执行

def func2():
    print("进入func2")
    gr1.switch()  # 挂起func2，切换到gr1
    print("回到func2")  # 不会执行（未被切换回来）

gr1 = greenlet(func1)
gr2 = greenlet(func2)
gr1.switch()  # 启动gr1
# 输出：
# 进入func1
# 进入func2
# 回到func1
```

#### Lua：`coroutine`库

* **特点**：Lua原生支持有栈协程，是语言核心特性，每个协程有独立的调用栈；  
* **原理**：通过`coroutine.create()`创建协程，`coroutine.resume()`恢复执行，`coroutine.yield()`挂起，可在任意位置`yield`；  

```lua
function task()
    print("任务开始")
    coroutine.yield()  -- 挂起，可在函数任意位置调用
    print("任务恢复")
end

co = coroutine.create(task)

print(coroutine.status(co))  -- 输出：suspended（未运行）

coroutine.resume(co)  -- 启动协程，输出：任务开始
print(coroutine.status(co))  -- 输出：suspended（挂起）

coroutine.resume(co)  -- 恢复协程，输出：任务恢复
print(coroutine.status(co))  -- 输出：dead（结束）
```

#### （3）Go：Goroutine（特殊的有栈协程）
- **特点**：Goroutine是Go的核心并发原语，本质是有栈协程，但由Go runtime而非用户手动调度；  
- **特殊之处**：  
  - 栈可动态增长（初始仅2KB，按需扩展至GB级），避免固定栈大小的限制；  
  - 由Go runtime的M:N调度器（映射到内核线程）自动调度，兼顾灵活性和性能；  
- **示例**：
  ```go
  package main

  import (
      "fmt"
      "time"
  )

  func task(id int) {
      for i := 0; i < 3; i++ {
          fmt.Printf("协程%d: %d\n", id, i)
          time.Sleep(100 * time.Millisecond)  // 模拟IO等待，主动让渡CPU
      }
  }

  func main() {
      go task(1)  // 启动协程1
      go task(2)  // 启动协程2
      time.Sleep(1 * time.Second)  // 等待协程完成
  }
  ```
  - 关键点：Goroutine可在任意位置（如`Sleep`、通道操作）被调度器挂起，无需显式`yield`，是“隐式有栈协程”的典型。


### 2. 无栈协程的实践
#### （1）C++20：`std::coroutine`
- **特点**：C++20引入无栈协程，完全由编译器实现（无运行时依赖），通过状态机转换协程函数；  
- **原理**：编译器将含`co_await`/`co_yield`的函数转换为包含状态变量、局部变量的结构体，挂起时保存状态，恢复时读取状态；  
- **示例**（简化版）：
  ```cpp
  #include <iostream>
  #include <coroutine>

  // 协程返回类型（需满足coroutine_traits要求）
  struct Task {
      struct promise_type {
          Task get_return_object() { return {}; }
          std::suspend_never initial_suspend() { return {}; }
          std::suspend_never final_suspend() noexcept { return {}; }
          void return_void() {}
          void unhandled_exception() {}
      };
  };

  Task counter() {
      for (int i = 0; i < 3; ++i) {
          std::cout << "计数: " << i << std::endl;
          co_await std::suspend_always{};  // 挂起点（仅能在此处挂起）
      }
  }

  int main() {
      auto task = counter();
      // 需手动驱动协程（实际中由调度器管理）
      std::cout << "恢复1\n";
      task.resume();  // 输出：计数: 0
      std::cout << "恢复2\n";
      task.resume();  // 输出：计数: 1
      return 0;
  }
  ```
  - 关键点：仅能在`co_await`处挂起，嵌套函数中若没有`co_await`则无法挂起，体现无栈协程的挂起点限制。


#### （2）C#：`async/await`
- **特点**：C# 5.0引入`async/await`，是典型的无栈协程，由编译器将`async`函数转换为状态机；  
- **原理**：`await`关键字标记挂起点，编译器生成包含状态、局部变量的类，通过状态切换实现协程的挂起和恢复；  
- **示例**：
  ```csharp
  using System;
  using System.Threading.Tasks;

  class Program {
      static async Task<int> FetchData() {
          Console.WriteLine("开始请求数据");
          // 仅能在await处挂起（等待异步操作）
          await Task.Delay(1000);  // 模拟网络请求
          Console.WriteLine("数据请求完成");
          return 42;
      }

      static async Task Main() {
          var task = FetchData();
          Console.WriteLine("等待数据中...");
          int result = await task;  // 恢复协程并获取结果
          Console.WriteLine($"结果: {result}");
      }
  }
  ```
  - 关键点：`FetchData`只能在`await`处挂起，若在普通函数（非`async`）中调用则无法挂起，依赖显式标记的挂起点。


#### （3）JavaScript：`async/await`
- **特点**：ES2017引入`async/await`，基于生成器（Generator）实现，本质是无栈协程；  
- **原理**：`async`函数返回Promise，`await`将函数拆分为多个“微任务”，通过事件循环调度，挂起点固定在`await`处；  
- **示例**：
  ```javascript
  async function fetchData() {
      console.log("开始请求");
      await new Promise(resolve => setTimeout(resolve, 1000));  // 挂起点
      console.log("请求完成");
      return 42;
  }

  (async () => {
      const task = fetchData();
      console.log("等待中...");
      const result = await task;  // 恢复并获取结果
      console.log(`结果: ${result}`);
  })();
  ```
  - 关键点：`await`是唯一挂起点，无法在普通函数中挂起，由JavaScript引擎的事件循环自动调度恢复。


#### （4）Python：`asyncio`（原生无栈协程）
- **特点**：Python 3.5+引入`async/await`，基于生成器实现无栈协程，需配合`asyncio`库调度；  
- **原理**：`async`函数被编译为生成器，`await`触发状态切换，上下文保存在生成器的`gi_frame`中，无独立栈；  
- **示例**：
  ```python
  import asyncio

  async def task():
      print("任务开始")
      await asyncio.sleep(1)  # 仅能在await处挂起（模拟IO）
      print("任务恢复")
      return "完成"

  async def main():
      t = task()
      print("等待任务...")
      result = await t  # 恢复任务
      print(result)

  asyncio.run(main())
  ```
  - 关键点：与`greenlet`（有栈）不同，`asyncio`协程不能在普通函数中挂起，必须通过`await`显式标记挂起点。

### 如何选择：场景决定类型

1. **优先有栈协程的场景**：  
   - 逻辑复杂，需要在嵌套函数中频繁挂起（如游戏AI、状态机）；  
   - 需兼容旧代码，不希望修改函数签名（有栈协程可通过库实现，对代码侵入小）；  
   - 通用并发任务（如多任务调度器）。  

2. **优先无栈协程的场景**：  
   - IO密集型任务（如网络爬虫、API服务），挂起点固定（仅在等待IO时挂起）；  
   - 对内存开销敏感（如百万级协程场景，无栈协程的内存优势明显）；  
   - 语言原生支持（如C#、JavaScript，用无栈协程更符合语言习惯）。  

有栈协程和无栈协程是协程的两种实现范式：

- 有栈协程以“独立栈”换取灵活性，支持任意位置挂起，但内存开销较高；  
- 无栈协程以“状态机”换取高效性，挂起点受限，但内存更省、切换更快。  

编程语言的选择往往反映了其设计权衡：系统级语言（如C++、Rust）倾向于无栈协程（性能优先），脚本语言（如Python、Lua）则同时支持两种（灵活性与易用性兼顾），而Go的Goroutine则融合了有栈协程的灵活性和动态栈的高效性，成为并发编程的标杆。理解两者的差异，才能在实际开发中选择最合适的协程模型。

## 13. 使用协程进行异步编程

非性能话题，本质上是协程特性的解读

## 14. 并行算法

### 算法并行化的一般思路

将普通C++算法并行化的核心是**拆分独立任务+利用多核资源**，但并非所有算法都适合并行化。

并行化存在“ overhead 开销”（线程创建、任务调度、数据同步），只有当“并行加速收益 > 开销”时才值得实施。以下准则可快速判断：

核心准则，Amdahl 定律（并行化上限）：并行加速比 \( S_p \) 满足：  
\[ S_p = \frac{1}{(1 - f) + \frac{f}{p}} \]  
- \( f \)：算法中“可并行部分的比例”（如循环体纯计算，无依赖 → \( f≈1 \)）；  
- \( p \)：CPU核心数（如8核CPU → \( p=8 \)）。  
- 结论：  
  - 若 \( f=0.9 \)（90%可并行），8核加速比≈5.7（理论上限）；  
  - 若 \( f<0.5 \)（仅50%可并行），8核加速比≈1.6（收益有限，不建议并行）。

| 场景                | 适合并行化                          | 不适合并行化                          |
|---------------------|-------------------------------------|---------------------------------------|
| 计算/通信比         | 计算密集（如矩阵运算、排序），计算量 > 1ms | IO密集（如文件读写、网络请求），计算量 < 100μs |
| 任务独立性          | 任务间无依赖（如独立数据块处理）    | 强依赖（如递归依赖、循环依赖）        |
| 数据规模            | 数据量≥10^4（如数组长度≥1e4）       | 数据量<1e3（并行开销 > 计算收益）      |
| 同步开销            | 无需同步或轻量同步（如原子操作）    | 频繁同步（如锁竞争严重、全局变量修改）|

工程时间阈值（经验值）

- 单线程执行时间 ≥ 500μs：并行化收益显著（如1ms任务，8核可降至~125μs）；  
- 单线程执行时间 < 100μs：不建议并行（线程创建+调度开销可能超过100μs）。

C++11及以上提供 `std::thread`、`std::async`、`std::execution`（C++17）等并行工具，以下案例按“实现难度”从低到高排序：

#### 方案1：循环并行（最常用，适合无依赖循环）

- 循环体无数据依赖（如数组元素独立计算、批量处理任务）；  
- 典型案例：数组平方、向量点积、批量数据过滤。

实现工具：

- C++17+：`std::for_each` + `std::execution::par`（最简单，自动调度线程）；  
- C++11/14：`std::thread` 手动拆分循环（兼容旧标准）。

数组元素平方（并行化改造）原始串行代码:

```cpp
#include <vector>
#include <iostream>

void serialSquare(const std::vector<int>& in, std::vector<int>& out) {
    for (size_t i = 0; i < in.size(); ++i) {
        out[i] = in[i] * in[i];  // 无依赖：每个元素独立计算
    }
}

int main() {
    std::vector<int> in(1e6, 2);  // 100万元素，值均为2
    std::vector<int> out(1e6);
    serialSquare(in, out);
    return 0;
}
```

并行化改造（C++17+，最简方案）：

```cpp
#include <vector>
#include <algorithm>  // std::for_each
#include <execution>  // std::execution::par

void parallelSquare(const std::vector<int>& in, std::vector<int>& out) {
    // std::execution::par：并行执行，自动利用多核
    std::for_each(std::execution::par, in.begin(), in.end(), 
        [&](int val) {
            size_t idx = &val - &in[0];  // 计算当前元素索引
            out[idx] = val * val;
        }
    );
}
```

并行化改造（C++11/14，兼容旧标准）,手动拆分循环到多个线程，避免线程创建过多（建议线程数=CPU核心数）：

```cpp
#include <vector>
#include <thread>
#include <functional>  // std::bind

void parallelSquareC11(const std::vector<int>& in, std::vector<int>& out) {
    size_t n = in.size();
    size_t threadNum = std::thread::hardware_concurrency();  // 获取CPU核心数（如8）
    std::vector<std::thread> threads;

    auto worker = [&](size_t start, size_t end) {
        for (size_t i = start; i < end; ++i) {
            out[i] = in[i] * in[i];
        }
    };

    size_t chunkSize = n / threadNum;  // 每个线程处理的元素数
    for (size_t i = 0; i < threadNum; ++i) {
        size_t start = i * chunkSize;
        size_t end = (i == threadNum - 1) ? n : (i + 1) * chunkSize;  // 最后一个线程处理剩余元素
        threads.emplace_back(worker, start, end);
    }

    // 等待所有线程完成
    for (auto& t : threads) {
        t.join();
    }
}
```

- 避免循环体中修改全局变量（需加锁，导致性能下降）；  
- 拆分区间时确保无重叠（如每个线程处理独立索引范围）。

#### 方案2：分治并行（适合递归算法，如排序、查找）

- 算法可拆分为“子问题”（子问题独立，无依赖）；  
- 典型案例：快速排序、归并排序、二分查找（大规模数据）、矩阵乘法。

实现工具：

- `std::async`（自动创建线程，返回`std::future`获取结果）；  
- 递归拆分+线程池（避免递归创建过多线程）。

并行归并排序（分治并行改造），归并排序的核心是“拆分+合并”：拆分后的子数组可并行排序，最后合并结果。

原始串行归并排序：

```cpp
#include <vector>
#include <algorithm>

void merge(std::vector<int>& arr, size_t left, size_t mid, size_t right) {
    std::vector<int> temp(right - left + 1);
    size_t i = left, j = mid + 1, k = 0;
    while (i <= mid && j <= right) {
        temp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];
    }
    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];
    std::copy(temp.begin(), temp.end(), arr.begin() + left);
}

void serialMergeSort(std::vector<int>& arr, size_t left, size_t right) {
    if (left >= right) return;
    size_t mid = left + (right - left) / 2;
    serialMergeSort(arr, left, mid);    // 左半部分排序
    serialMergeSort(arr, mid + 1, right);  // 右半部分排序
    merge(arr, left, mid, right);       // 合并
}
```

并行化改造（用`std::async`分治）：

```cpp
#include <vector>
#include <algorithm>
#include <future>  // std::async, std::future

void parallelMergeSort(std::vector<int>& arr, size_t left, size_t right) {
    const size_t THRESHOLD = 1e4;  // 阈值：子数组长度<1e4时串行（避免小任务并行开销）
    if (right - left < THRESHOLD) {
        serialMergeSort(arr, left, right);  // 小数据串行
        return;
    }

    size_t mid = left + (right - left) / 2;
    // 异步执行左半部分排序，返回future（不阻塞当前线程）
    auto futureLeft = std::async(std::launch::async, 
        parallelMergeSort, std::ref(arr), left, mid);
    // 当前线程执行右半部分排序
    parallelMergeSort(arr, mid + 1, right);
    // 等待左半部分完成
    futureLeft.get();
    // 合并结果
    merge(arr, left, mid, right);
}
```

- 设置“串行阈值”：子问题规模过小时（如<1e4），串行执行比并行更高效；  
- 避免递归创建过多线程：`std::async`默认使用线程池（C++17后），但旧标准可能创建大量线程，需手动控制线程数。

#### 方案3：数据并行（适合大规模数据处理，如滤波、统计）

- 对大规模数据执行相同操作（如求和、均值、滤波、去重）；  
- 典型案例：数组求和、图像像素处理、日志统计。

实现工具：

- C++17+：`std::reduce`（并行求和，替代`std::accumulate`）；  
- 自定义线程池+任务队列（适合高频数据处理）。

并行数组求和（数据并行改造），原始串行代码：

```cpp
#include <vector>
#include <numeric>  // std::accumulate

int serialSum(const std::vector<int>& arr) {
    // 串行求和：O(n)时间，单线程
    return std::accumulate(arr.begin(), arr.end(), 0);
}
```

并行化改造（C++17+，`std::reduce`）：

```cpp
#include <vector>
#include <numeric>  // std::reduce
#include <execution>  // std::execution::par

int parallelSum(const std::vector<int>& arr) {
    // 并行求和：自动拆分数据到多核，无锁同步
    return std::reduce(std::execution::par, 
        arr.begin(), arr.end(), 0);
}
```

并行化改造（C++11/14，线程池求和）：自定义简单线程池，避免重复创建线程（适合高频调用场景）：

```cpp
#include <vector>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>

// 简单线程池
class ThreadPool {
public:
    explicit ThreadPool(size_t threadNum) {
        for (size_t i = 0; i < threadNum; ++i) {
            threads.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx);
                        cv.wait(lock, [this]() { return stop || !tasks.empty(); });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mtx);
            stop = true;
        }
        cv.notify_all();
        for (auto& t : threads) t.join();
    }

    template <typename F>
    void enqueue(F&& f) {
        {
            std::unique_lock<std::mutex> lock(mtx);
            tasks.emplace(std::forward<F>(f));
        }
        cv.notify_one();
    }

private:
    std::vector<std::thread> threads;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;
};

// 并行求和（基于线程池）
int parallelSumC11(const std::vector<int>& arr) {
    size_t n = arr.size();
    size_t threadNum = std::thread::hardware_concurrency();
    ThreadPool pool(threadNum);
    std::vector<int> partialSums(threadNum, 0);  // 每个线程的部分和

    size_t chunkSize = n / threadNum;
    for (size_t i = 0; i < threadNum; ++i) {
        size_t start = i * chunkSize;
        size_t end = (i == threadNum - 1) ? n : (i + 1) * chunkSize;
        pool.enqueue([&, i, start, end]() {
            for (size_t j = start; j < end; ++j) {
                partialSums[i] += arr[j];  // 每个线程计算部分和（无锁，独立数组）
            }
        });
    }

    // 汇总部分和
    return std::accumulate(partialSums.begin(), partialSums.end(), 0);
}
```

- 避免共享变量竞争：如用“部分和数组”（每个线程写独立元素）替代全局变量求和（需加锁）；  
- 线程池适合高频场景：若仅调用1次，`std::async`更简洁；若频繁调用，线程池可减少线程创建开销。

#### 并行化避坑指南（常见问题与解决方案）

| 常见问题                | 解决方案                                  |
|-------------------------|-------------------------------------------|
| 锁竞争导致性能下降      | 1. 避免共享变量；2. 用原子操作（`std::atomic`）替代锁；3. 拆分数据到线程本地 |
| 线程创建过多（资源耗尽）| 1. 线程数=CPU核心数；2. 用线程池；3. 设置串行阈值 |
| 数据依赖导致结果错误    | 1. 先梳理依赖关系（如循环依赖无法并行）；2. 拆分无依赖任务；3. 用同步机制（如`std::barrier`）控制执行顺序 |
| 缓存伪共享（False Sharing） | 1. 数据对齐（如每个线程的部分和数组元素按缓存行对齐）；2. 用`std::hardware_destructive_interference_size`（C++17） |

### 并行标准库算法

1. **定义**：`std::execution::execution_policy_tag_t` 是C++17标准库中定义的**空类标签基类**（empty base class），位于`<execution>`头文件，命名空间为`std::execution`。
2. **核心使命**：作为所有标准执行策略的统一基类，为并行算法提供**执行方式的显式指定接口**，屏蔽底层线程管理细节，同时规范不同执行模式下的行为契约（如线程安全、迭代器兼容性）。
3. **关键特性**：
   - 无成员函数或数据成员，仅用于**类型标识**；
   - 支持`std::is_execution_policy`类型特性判断（用于编译期校验是否为合法执行策略）；
   - 所有标准执行策略（`seq`/`par`/`unseq` /`par_unseq`）均满足`std::is_execution_policy_v<Policy>`为`true`。

| 执行策略   | 英文全称  | 执行方式  | 迭代器最低要求   | 异常处理方式     | 线程安全特性   | 核心约束与优化支持 | 适用场景 |
|--------|---------|-------------|-----------|----------|------------|------------|-------------|
| `std::execution::seq`             | Sequential execution    | 单线程串行执行，严格遵循迭代器顺序        | 前向迭代器（ForwardIterator） | 异常正常传播（抛出后终止算法，无额外行为）                                     | 无多线程开销，仅单线程访问数据，天然无数据竞争                                 | 1. 严格保持迭代顺序；2. 无并行/向量化优化；3. 兼容所有符合前向迭代器的容器 | 小规模数据、强顺序依赖、计算量小（<100μs）、依赖迭代顺序的场景 |
| `std::execution::par`             | Parallel execution      | 多线程并行执行，同一线程内迭代顺序不变，不跨线程重排 | 双向迭代器（BidirectionalIterator） | 任意线程抛出未捕获异常时，调用 `std::terminate` 终止程序                       | 算法内部保证无数据竞争；用户提供的函数对象若修改共享状态，需手动保证线程安全（原子操作/锁） | 1. 保持线程内迭代顺序，不跨线程重排；2. 支持多线程并行，不支持向量化（SIMD）；3. 兼容双向迭代器及以上容器 | 大规模无依赖数据、需保证线程内迭代顺序、计算密集型（>500μs）、链表等双向迭代器容器场景 |
| `std::execution::par_unseq`       | Parallel unsequenced execution | 多线程并行执行，允许跨线程重排迭代，支持向量化 | 随机访问迭代器（RandomAccessIterator） | 任意线程抛出未捕获异常时，调用 `std::terminate` 终止程序                       | 算法内部保证无数据竞争；函数对象需支持“并行无序调用”（无副作用、不依赖迭代顺序） | 1. 允许跨线程重排迭代；2. 支持多线程并行+向量化（SIMD）优化；3. 仅兼容随机访问迭代器容器 | 超大规模数据、无顺序依赖、追求极致并行性能（如矩阵运算、数组批量处理） |
| `std::execution::unseq`           | Unsequenced execution   | 单线程执行，允许向量化（SIMD）优化，不保证迭代顺序(迭代顺序可能被重排以适配向量化指令。) | 随机访问迭代器（RandomAccessIterator） | 异常正常传播（抛出后终止算法，无额外行为）                                     | 单线程内并行（向量化），无多线程数据竞争风险                                   | 1. 单线程内允许重排迭代以适配向量化；2. 支持 SIMD 优化；3. 禁止修改非线程局部状态；4. 仅兼容随机访问迭代器容器 | 单线程场景下追求向量化优化、无顺序依赖的大规模数据处理（如数组滤波、数值计算） |

### 基于索引的for_each

标准库算法通过在库中包含算法std::for_each()提供了等效于基于范围的for循环。

基于索引的for循环可以通过将标准库算法std::for_each()与范围库中的std::views::iota()结合使用来创建。

```cpp
template <typename Policy, typename Index, typename F>
auto parallel_for(Policy&& p, Index first, Index last, F f) {
  auto r = std::views::iota(first, last);
  std::for_each(p, r.begin(), r.end(), std::move(f));
} 

auto v = std::vector<std::string>{"A", "B", "C"};
parallel_for(std::execution::par, size_t{0}, v.size(),
              & { v[i] += std::to_string(i + 1); }); 
```

## 15. 其他书籍推荐
