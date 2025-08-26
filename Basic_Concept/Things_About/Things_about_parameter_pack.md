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

参数包展开是指在一个后跟省略号（...）的模式中，当模式中至少出现一个参数包名称时，该模式会被展开为零个或多个模式实例。在展开过程中，**参数包的名称会按顺序被包中的每个元素替换：对齐说明符(Alignment specifier)的实例化以空格分隔、其他实例化以逗号分隔**

```cpp
template<class... Us>
void f(Us... pargs) {}
 
template<class... Ts>
void g(Ts... args)
{
    f(&args...); // “&args...” is a pack expansion
                 // “&args” is its pattern
}
 
g(1, 0.2, "a"); // Ts... args expand to int E1, double E2, const char* E3
                // &args... expands to &E1, &E2, &E3
                // Us... pargs expand to int* E1, double* E2, const char** E3
```

**如果同一个模式中出现两个参数包的名称，它们会被同时展开，且必须具有相同的长度**

```cpp
template<typename...>
struct Tuple {};
 
template<typename T1, typename T2>
struct Pair {};
 
template<class... Args1>
struct zip
{
    template<class... Args2>
    struct with
    {
        typedef Tuple<Pair<Args1, Args2>...> type;
        // Pair<Args1, Args2>... is the pack expansion
        // Pair<Args1, Args2> is the pattern
    };
};
 
typedef zip<short, int>::with<unsigned short, unsigned>::type T1;
// Pair<Args1, Args2>... expands to
// Pair<short, unsigned short>, Pair<int, unsigned int> 
// T1 is Tuple<Pair<short, unsigned short>, Pair<int, unsigned>>
 
// typedef zip<short>::with<unsigned short, unsigned>::type T2;
// error: pack expansion contains packs of different lengths
```

**如果一个参数包展开嵌套在另一个参数包展开之中，那么出现在最内层参数包展开中的参数包会被最内层的展开所展开，并且在最外层的参数包展开中必须提及另一个参数包，但该参数包不能出现在最内层的展开中。换句话说，外层的展开可以包含内层展开中未被展开的参数包**

```cpp
template<class... Args>
void g(Args... args)
{
    // 同时展开两个参数包 (Args 和 args)
    f(const_cast<const Args*>(&args)...); 
 
    // 嵌套的参数包展开：
    // 内层展开是 "args...", 先被展开
    // 外层展开是 h(args...) + args..., 后被展开
    // 结果为 h(E1, E2, E3) + E1, h(E1, E2, E3) + E2, h(E1, E2, E3) + E3
    f(h(args...) + args...); 
}
```

**当参数包中的元素数量为零（空参数包）时，参数包展开的实例化不会改变封闭构造的语法解释，即使完全省略参数包展开会导致格式错误或语法歧义。这种情况下，实例化会产生一个空列表**

```cpp
template<class... Bases> 
struct X : Bases... { };
 
template<class... Args> 
void f(Args... args) 
{
    X<Args...> x(args...);
}
 
// 合法：X<> 没有基类
// x 是 X<> 类型的变量，进行值初始化
template void f<>(); 
```

## 参数包展开示例

参数包展开可以应用在很多场景中，以下是一些举例

### 函数实参列表（Function argument lists）

参数包展开可以出现在函数调用操作符的括号内，在这种情况下，省略号左侧最大的表达式或花括号括起来的初始化列表是将被展开的模式

```cpp
f(args...);              // expands to f(E1, E2, E3)
f(&args...);             // expands to f(&E1, &E2, &E3)
f(n, ++args...);         // expands to f(n, ++E1, ++E2, ++E3);
f(++args..., n);         // expands to f(++E1, ++E2, ++E3, n);
 
f(const_cast<const Args*>(&args)...);
// f(const_cast<const E1*>(&X1), const_cast<const E2*>(&X2), const_cast<const E3*>(&X3))
 
f(h(args...) + args...); // expands to 
// f(h(E1, E2, E3) + E1, h(E1, E2, E3) + E2, h(E1, E2, E3) + E3)
```

### 函数形参列表（Function parameter lists）

函数参数列表可以使用省略号（...）来定义可变参数列表。这种参数包可以出现在模板函数中，用于处理不同数量和类型的参数。

```cpp
template<typename... Ts>
void f(Ts... args) {
    // args 是一个参数包，包含所有传递给 f 的参数
}
 
f('a', 1); // 调用时，Ts... 展开为 char 和 int，函数签名变为 void f(char, int)
f(0.1);    // 调用时，Ts... 展开为 double，函数签名变为 void f(double)

//============================================
template<typename... Ts, int... N>
void g(Ts (&...arr)[N]) {
    // Ts (&...arr)[N] 是一个参数包模式，表示多个数组引用参数
    // arr 是参数包，包含多个数组引用
    // N 是参数包，包含每个数组的大小
}
 
int n[1];
 
g<const char, int>("a", n); 
// 调用时，Ts (&...arr)[N] 展开为 const char (&)[2], int(&)[1]
// "a" 是一个 const char[2]（包含 '\0'），n 是一个 int[1]
```

在 `Ts (&...arr)[N]` 中，省略号是 最内层的元素，而不是像其他参数包展开中那样作为最后一个元素。这意味着展开时，`Ts` 和 `N` 会同时被展开

### 括号初始化（Parenthesized initializers）

包括直接初始化、函数风格的类型转换，以及其他类似函数调用表达式的场景（如成员初始化、new 表达式等），其规则与函数调用表达式中的参数包展开规则相同

```cpp
Class c1(&args...);             // calls Class::Class(&E1, &E2, &E3)
Class c2 = Class(n, ++args...); // calls Class::Class(n, ++E1, ++E2, ++E3);
 
::new((void *)p) U(std::forward<Args>(args)...) // std::allocator::allocate
```

### 初始化列表（Brace-enclosed initializers）

```cpp
template<typename... Ts>
void func(Ts... args)
{
    const int size = sizeof...(args) + 2;
    int res[size] = {1, args..., 2};
 
    // since initializer lists guarantee sequencing, this can be used to
    // call a function on each element of a pack, in order:
    int dummy[sizeof...(Ts)] = {(std::cout << args, 0)...};
}
```

### 模板实例化参数列表（Template argument lists）

```cpp
template<class A, class B, class... C>
void func(A arg1, B arg2, C... arg3)
{
    container<A, B, C...> t1; // expands to container<A, B, E1, E2, E3> 
    container<C..., A, B> t2; // expands to container<E1, E2, E3, A, B> 
    container<A, C..., B> t3; // expands to container<A, E1, E2, E3, B> 
}
```

### 模板形参列表（Template parameter list）

```cpp
template<typename... T>     // typename... T 定义了一个模板参数包 T，它可以接受任意数量的类型参数
struct value_holder
{
    // 这里的 T... 展开为一个模板参数列表，其中每个参数都是外层模板参数包 T 中的一个类型, 
    // 参数包展开生成常量模板参数列表
    template<T... Values> 
    struct apply {};      // 例如：<int, char, int(&)[5]>
};
```

### 基类多继承

可变模板参数包来支持多个基类的继承，并在构造函数的成员初始化列表中使用参数包展开调用每个基类的构造函数

```cpp
template<class... Mixins>
class X : public Mixins...
{
public:
    X(const Mixins&... mixins) : Mixins(mixins)... {}
};
```

### lambda表达式捕获组

```cpp
template<class... Args>
void f(Args... args)
{
    auto lm = [&, args...] { return g(args...); };
    lm();
}
```

### `sizeof...`

sizeof...(Types)操作符用于计算Types参数包中的类型个数

```cpp
template<class... Types>
struct count
{
    static const std::size_t value = sizeof...(Types);
};
```

### 动态异常（Until C++17）

```cpp
template<class... X>
void func(int arg) throw(X...)
{
    // ... 在不同情况下抛出不同的X类型异常
}
```

现代 C++ 推荐使用`noexcept`来指定函数是否可能抛出异常，`noexcept`仅表明函数是否可能抛出异常（是一个布尔属性），而动态异常规范可以指定具体的异常类型，但这一特性因实现复杂且实用性有限而被移除。

### 内存对齐（Alignment Specifier）

在 C++ 中，`alignas`关键字用于指定变量、类型或对象的对齐要求。当与模板参数包结合使用时，alignas支持包展开（pack expansion）。常规情况下，`alignas`可以接受两种形式的参数：类型：如`alignas(int)`；常量表达式：如`alignas(16)`表示 16 字节对齐

在模板定义中，alignas可以使用包展开语法，将参数包中的每个元素作为单独的对齐说明符。

```cpp
// 模板定义
template<class... T>
struct Align
{
    alignas(T...) unsigned char buffer[128];
};

// 实例化目标
Align<int, short> a;

// 展开形式：注意展开后的对齐说明符之间是空格分隔，而非逗号
alignas(int) alignas(short) unsigned char buffer[128];
```

展开后的多个`alignas`说明符表示该对象需要满足所有指定的对齐要求，即实际对齐值将是所有指定对齐值中的最大值。

## 折叠表达式（[Fold expressions (since C++17)](https://en.cppreference.com/w/cpp/language/fold.html)）

> Reduces ([folds](https://en.wikipedia.org/wiki/Fold_(higher-order_function))) a pack over a binary operator.

在二元运算符上折叠参数包，**折叠表达式必须包含在括号内**，包含四种语法形式

```cpp
// 一元右折叠，Unary right fold (E op ...) becomes (E1 op (... op (EN-1 op EN)))
( pack op ... )

// 一元左折叠，Unary left fold (... op E) becomes (((E1 op E2) op ...) op EN)
(... op pack)

// 二元右折叠，Binary right fold (E op ... op I) becomes (E1 op (... op (EN−1 op (EN op I))))
(pack op ... op init)

// 二元左折叠，Binary left fold (I op ... op E) becomes ((((I op E1) op E2) op ...) op EN)
(init op ... op pack)
```

* ops支持32种二元运算符，包括`+`、`-`、`*`、`/`、`%`、`^`、`&`、`|`、`=`、`<`、`>`、`<<`、`>>`、`+=`、`-=`、`*=`、`/=`、`%=`、`^=`、`&=`、`|=`、`<<=`、`>>=`、`==`、`!=`、`<=`、`>=`、`&&`、`||`、`,`、`.*`、`->*`。二元折叠中两个op必须相同。
* pack：含未展开参数包的表达式，顶层不含优先级低于 cast 的运算符（即 cast-expression）。
* init：不含未展开参数包的表达式，顶层不含优先级低于 cast 的运算符（即 cast-expression）。
* 一元折叠中参数包长度为 0 时，仅允许以下运算符：`&&`：结果为`true` `||`：结果为`false` `,`：结果为`void()`

## lambda捕获组中的参数包（[Lambda capture](https://en.cppreference.com/w/cpp/language/lambda.html#Lambda_capture)）

语法结构

```cpp
... identifier initializer      // by-copy capture with an initializer that is a pack expansion

& ... identifier initializer    // by-reference capture with an initializer that is a pack expansion
```

## 结构化绑定参数包（[Structured binding declaration (since C++17)](https://en.cppreference.com/w/cpp/language/structured_binding.html)）

> [Structured Binding Upgrades in C++26](https://biowpn.github.io/bioweapon/2024/12/03/structured-bindings-cpp26.html)
>
> [Structured Bindings can introduce a Pack](https://isocpp.org/files/papers/P1061R10.html)

结构化绑定(在C++17倍引入)：用于将聚合类型（数组、结构体、std::pair、std::tuple等）的成员 “解包” 到变量中，简化对复合类型的访问

```cpp
auto [a, b] = std::make_pair(1, "hello"); // 解包pair，a=1，b="hello"
int arr[3] = {10, 20, 30};
auto [x, y, z] = arr; // 解包数组，x=10，y=20，z=30
```

P1061 提案的核心是通过结构化绑定语法auto [...xs] = p;，将聚合类型（如结构体、数组等）的成员转换为参数包，从而简化泛型编程中对复合类型成员的批量处理。

* 无缝转换结构体为参数包：例如通过以下函数可将任意类型T的成员转换为std::tie形式的参数包，为反射场景提供基础：

```cpp
template <class T>
auto tie_as_tuple(T& x) {
    auto& [...xs] = x; // 结构化绑定生成参数包xs...
    return std::tie(xs...);
}
```

* 适用范围广泛：不仅支持结构体（聚合类型），还可通过特化std::tuple_size、std::tuple_element及提供get函数，扩展到其他类类型，无需依赖复杂的 hacks
  
1061 最初的目标更为宏大：在非模板场景中也能通过结构化绑定引入参数包，例如：

```cpp
struct Point { int x, y; };
int main() {
    Point p{3, 4};
    auto [...cords] = p; // 尝试在main（非模板）中直接生成参数包cords...
    auto dist_sq = (cords * cords + ...); // 配合折叠表达式使用
}
```

为实现这一目标，提案曾引入 **“隐式模板区域（implicit template region）”** 概念：在非模板场景中使用结构化绑定包时，自动将附近代码视为 “类模板区域”，以支持参数包语法。但这一策略最终因实现复杂性（如编译器处理逻辑、兼容性等问题）在 R9 版本后被放弃。

由于上述挑战，最终纳入 C++26 的是简化版 P1061：结构化绑定仅能在模板中引入参数包，非模板场景中直接使用会报错：

```cpp
// 非模板场景直接使用（错误）
struct Point { int x, y; };
int main() {
    Point p{3, 4};
    auto [...cords] = p; // 不允许
}
```

但非模板场景可通过显式引入模板区域间接实现，例如用 lambda 创建局部模板上下文：

```cpp
struct Point { int x, y; };
int main() {
    Point p{3, 4};
    // 通过lambda的模板参数（隐式auto）创建模板区域
    [&](auto& _p) {
        auto [...cords] = _p; // 合法，cords...为参数包
    }(p);
}
```

这与 “隐式模板区域” 的设计思路类似，只是需要手动触发。

## 参数包访问（[Pack indexing (since C++26)](https://en.cppreference.com/w/cpp/language/pack_indexing.html)）

参数包索引（Pack Indexing）是C++26引入的新特性，允许通过编译时索引直接访问参数包（parameter pack）中的元素，语法类似数组索引。这一特性简化了模板中对参数包特定元素的访问，无需依赖递归模板或辅助元函数。

参数包索引允许在模板中通过`pack[index]`的形式访问参数包中的第`index`个元素（索引从0开始）。其支持两种参数包：

- **类型参数包（type pack）**：如`Ts...`，通过`Ts[N]`访问第`N`个类型；
- **值参数包（value pack）**：如`args...`，通过`args[N]`访问第`N`个值。

**语法规则**：

- 索引`N`必须是编译时常量表达式（`constexpr`）；
- 索引范围为`[0, sizeof...(pack) - 1]`，超出范围会导致编译错误；
- 参数包不能为空（除非索引在有效范围内，但空包无元素，因此索引必然无效）。

下面是一个类型参数包索引的例子：

```cpp
template <typename... Ts>
struct TypePackExample {
    // 访问第0个类型（第一个类型）
    using FirstType = Ts[0];
    // 访问第2个类型（第三个类型，若参数包长度≥3）
    using ThirdType = Ts[2];
};

// 实例化：Ts = [int, double, std::string]
using MyPack = TypePackExample<int, double, std::string>;
static_assert(std::is_same_v<MyPack::FirstType, int>);         // 正确
static_assert(std::is_same_v<MyPack::ThirdType, std::string>); // 正确
```

值参数包索引用于访问函数或模板中值参数包的特定元素，可直接获取参数包中第`N`个值。

```cpp
template <auto... Args>
constexpr auto get_value_at(size_t index) {
    // 通过索引返回参数包中的值（编译时确定）
    return Args[index];
}

constexpr auto val0 = get_value_at<10, 20, 30>(0); // val0 = 10
constexpr auto val2 = get_value_at<10, 20, 30>(2); // val2 = 30
```

可与折叠表达式、结构化绑定等结合使用，例如从参数包中提取特定元素组成新包：

```cpp
template <typename... Ts>
auto extract_pair(Ts&&... args) {
    return std::make_pair(args[0], args[1]); // 提取前两个元素
}
```

## 参数包的调试技巧

1. **打印参数包大小和类型**

```cpp
template <typename... Args>
void debug_args(Args&&... args) {
    std::cout << "参数包大小: " << sizeof...(args) << std::endl;
    
    std::cout << "参数类型: ";
    ((std::cout << typeid(args).name() << " "), ...);
    std::cout << std::endl;
}
```

2. **分步调试复杂的参数包展开**

```cpp
template <typename T>
void debug_single(T&& arg) {
    std::cout << "单个参数: " << arg << " (类型: " << typeid(arg).name() << ")" << std::endl;
}

template <typename... Args>
void debug_step_by_step(Args&&... args) {
    (debug_single(std::forward<Args>(args)), ...);
}
```

## 一些示例

### 内存池

在一个需要频繁分配小对象的系统中，我们使用参数包实现了高效的内存池：

**优化前**：标准分配器

```cpp
template <typename T>
T* create() {
    return new T();
}

template <typename T, typename Arg>
T* create(Arg&& arg) {
    return new T(std::forward<Arg>(arg));
}

template <typename T, typename Arg1, typename Arg2>
T* create(Arg1&& arg1, Arg2&& arg2) {
    return new T(std::forward<Arg1>(arg1), std::forward<Arg2>(arg2));
}

// 更多重载...
```

**优化后**：基于参数包的内存池

```cpp
template <typename T, typename... Args>
T* create(Args&&... args) {
    void* memory = memory_pool_.allocate(sizeof(T), alignof(T));
    
    try {
        return new(memory) T(std::forward<Args>(args)...);
    } catch (...) {
        memory_pool_.deallocate(memory);
        throw;
    }
}
```