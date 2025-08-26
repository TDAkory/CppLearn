# C++高性能编程

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

详见 [C++20 Additional synchronization primitives]()