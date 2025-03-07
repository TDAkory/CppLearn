# C++高性能编程

> [C++ 高性能编程（全）](https://www.cnblogs.com/apachecn/p/18172912)

**零开销原则**：你不使用的东西，你就不需要付费；你使用的东西，你无法手工编码得更好

一些 C++ 的旧但强大的特性，这些特性与健壮性有关，而不是性能，很容易被忽视：值语义、const正确性、所有权、确定性销毁和引用。

> 在内部，引用是一个不允许为空或重新指向的指针；因此，当将其传递给函数时不涉及复制。

**C++的缺点**：长时间的编译时间和导入库的复杂性（直到 C++20，C++一直依赖于一个过时的导入系统）；缺乏提供的库（C++提供的几乎只是最基本的算法、线程，以及从 C++17 开始的文件系统处理，图形、用户界面、网络、线程、资源处理等只能依赖外部库）

## 基本C++技术

### `auto`自动类型推到

**在函数签名中使用 `auto`**

| 显式类型的传统语法 | 使用 `auto` 的新语法 |
|--------------------|----------------------|
| `int val() const { return m_; }` | `auto val() const { return m_; }` |
| `const int& cref() const { return m_; }` | `auto& cref() const { return m_; }` |
| `int& mref() { return m_; }` | `auto& mref() { return m_; }` |

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

## 分析和测量性能

**摊销时间复杂度**：摊销时间复杂度关注的是一系列操作的总时间，然后将其平均分配到每个操作上。即使某些操作的时间复杂度较高，只要这些操作的发生频率较低，整体平均性能仍然可以很好。

**如何优化**：明确目标、测量、找出瓶颈、做出合理猜测、优化、评估、重构

**性能特性**：延迟/响应时间、吞吐量、IO bound、CPU bound、功耗
