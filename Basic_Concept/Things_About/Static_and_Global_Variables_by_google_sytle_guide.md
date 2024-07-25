# Google风格中关于静态和全局变量的建议

> 译自[Static and Global Variables](https://google.github.io/styleguide/cppguide.html#Static_and_Global_Variables)

通常不建议使用具有静态生命周期（[static storage duration](https://en.cppreference.com/w/cpp/language/storage_duration#Storage_duration)）的对象，除非它们是平凡可析构（[trivially destructible](https://en.cppreference.com/w/cpp/types/is_destructible)）的。简单来说，平凡可析构意味着析构函数不执行任何操作，即使考虑对象的成员和对象的基类析构函数也是如此。更正式地说，这意味着该类型没有 **用户定义的** 或 **虚的** 析构函数，并且所有基类和非静态成员都是平凡可析构的。

函数的静态局部变量可以使用动态初始化。但是，不建议对 **类的静态成员变量** 或 **命名空间范围内的变量** 使用动态初始化，一个例外情况是：如果一个全局变量的声明，单独考虑，可以是constexpr。

## 定义

每个对象都有一个存储持续时间，这与其生命周期相关。具有静态生命周期的对象从其初始化点一直存在到程序结束。这类对象表现为命名空间范围内的变量（“全局变量”）、类的静态数据成员 、使用 static 说明符声明的函数局部变量。函数局部静态变量在流程首次执行到其声明时进行初始化；所有其他具有静态生命周期的对象在程序启动时作为一部分进行初始化。所有具有静态生命周期的对象在程序退出时被销毁（这一步是在 **unjoined线程** 被销毁之前发生的）。

## 优势

全局变量和静态变量对于很多应用场景都非常有用：命名常量、某些编译单元内部的辅助数据结构、命令行标志、日志记录、注册机制、后台基础设施等。

## 劣势

使用动态初始化或具有非平凡析构函数的全局变量和静态变量会产生复杂性，这很容易导致难以发现的错误。动态初始化在不同的翻译单元之间没有顺序，析构也没有顺序（只能保证析构以与初始化相反的顺序发生）。当一个初始化引用具有静态生命周期的另一个变量时，可能会导致在对象的生存期开始之前（或结束之后）访问该对象。此外，如果程序启动在退出时未连接的线程，并且如果其析构函数已经运行，这些线程可能会尝试在对象的生存期结束后访问对象。

## 关于析构的决策

当析构函数是平凡的时，它们的执行根本不受顺序的约束（它们实际上不会“运行”）；否则，我们就会面临在对象的生存期结束后访问对象的风险。因此，我们只允许具有平凡可析构性的具有静态生命周期的对象。基本类型（如指针和整数）是平凡可析构的，平凡可析构类型的数组也是如此。请注意，用 `constexpr` 标记的变量是平凡可析构的。 

```c
const int kNum = 10;  // 允许

struct X { int n; };
const X kX[] = {{1}, {2}, {3}};  // 允许

void foo() {
  static const char* const kMessages[] = {"hello", "world"};  // 允许
}

// 允许: constexpr 表达式能够保证其修饰的对象是可以平凡析构的
constexpr std::array<int, 3> kArray = {1, 2, 3};
```

```c
// 不推荐: non-trivial destructor
const std::string kFoo = "foo";

// Bad for the same reason, even though kBar is a reference 
// (规则对于生命周期扩展的临时对象也有效).
const std::string& kBar = StrCat("a", "b", "c");

void bar() {
  // 不推荐: non-trivial destructor.
  static std::map<int, int> kData = {{1, 0}, {2, 0}, {3, 0}};
}
```

## 关于构造的决策

初始化是一个更为复杂的话题，因为我们不仅必须考虑类构造函数是否执行，还必须考虑到初始化器的执行

```c
int n = 5;    // Fine
int m = f();  // ? (Depends on f)
Foo x;        // ? (Depends on Foo::Foo)
Bar y = g();  // ? (Depends on g and on Bar::Bar)
```

上面这个例子中，除了第一条语句外，其他语句都存在不确定的初始化顺序。

我们需要关注的概念在 C++ 标准中被称为常量初始化（constant initialization）。这意味着初始化表达式是一个常量表达式，如果对象是通过构造函数调用进行初始化的，那么构造函数也必须被指定为`constexpr`，比如如下示例：

```c
struct Foo { constexpr Foo(int) {} };

int n = 5;  // Fine, 5 is a constant expression.
Foo x(2);   // Fine, 2 is a constant expression and the chosen constructor is constexpr.
Foo a[] = { Foo(1), Foo(2), Foo(3) };  // Fine
```

常量初始化始终是推荐的。具有静态生命周期的变量的常量初始化应当用 `constexpr` 或 `constinit` 进行标记。任何未如此标记的非局部静态生命周期变量应当被假定为具有动态初始化，并且需要非常仔细地审查。

相对的，下面这些初始化可能存在问题：

```c
// Some declarations used below.
time_t time(time_t*);      // Not constexpr!
int f();                   // Not constexpr!
struct Bar { Bar() {} };

// Problematic initializations.
time_t m = time(nullptr);  // Initializing expression not a constant expression.
Foo y(f());                // Ditto
Bar b;                     // Chosen constructor Bar::Bar() not constexpr.
```

不建议对非局部变量进行动态初始化，通常情况下是禁止的。然而，如果程序的任何方面都不依赖于 该初始化与所有其他初始化 的顺序，我们确实允许这样做。在这些限制下，初始化的顺序不会产生可观察到的差异。例如：

```c
int p = getpid();  // Allowed, as long as no other static variable
                   // uses p in its own initialization.
```

静态局部变量的动态初始化是被允许的（并且常见）。

## 一些通用的建议

* 对于全局或静态字符串常量，应使用constexpr变量指向字符串字面量、字符数组或字符指针，以确保它们具有静态存储期限。字符串字面量已经具有静态存储期限，通常足够用。参见[TotW #140](https://abseil.io/tips/140)。

* 静态变量不应使用标准库中的动态容器，因为它们有非平凡的析构函数。应使用简单数组或排序数组，对于小集合，线性搜索完全足够（并且由于内存局部性而高效）,此外也可以保持集合有序，来使用二分搜索算法。如果你确实更喜欢标准库的动态容器，请考虑使用函数局部静态指针，如下所述。

* 应避免使用智能指针作为静态或全局变量，因为它们在析构时执行清理，可能会引起问题。作为替代，可以使用普通指针并确保不删除分配的内存。

* 自定义类型的静态变量应具有平凡的析构函数和constexpr构造函数，以保证它们可以安全地作为静态变量使用。

* 如果以上方法都不可行，可以通过函数局部静态指针或引用来创建一个动态对象，但永不释放它，从而避免复杂的资源管理问题。（例如，`static const auto& impl = *new T(args...);`）
