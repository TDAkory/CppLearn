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

