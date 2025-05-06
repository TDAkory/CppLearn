# 为什么C++有参数依赖查找（ADL）？

从一个编译问题说起：

```shell
xxx.cc:100: error: reference to 'sort' is ambiguous
    sort(vec_.begin(), vec_.end(), std::less<double>());

yyy.h:5 note: candidate found by name lookup is 'sort'
namespace sort{
          ^
```

问题的来源，是在一个复杂项目的编译时，由于新引入的一个库的文件`xxx.cc:100`包含一句`sort`语句，报出了如上的编译错误。编译器发现有多个不同的`sort`名字候选，无法确定调用哪一个，按照编译器的提示，它首先找到的是一个位于`yyy.h:5`名为 `sort` 的命名空间。其中 `xxx.cc` 是库的源文件，而 `yyy.h` 是复杂项目自身的源文件。

这里引起了我们的兴趣：

* 编译器为什么会找到一个命名空间，什么是 name lookup ？
* 为什么库和复杂项目单独编译的时候都没有问题 ？

## 什么是 name lookup

> [Name lookup](https://en.cppreference.com/w/cpp/language/lookup)

按照定义，名称查找是这样一个过程：当程序中遇到一个名称时，将其与引入该名称的声明关联起来。它确保了代码中的每个名称都能正确地关联到其声明。这个过程包括非限定名称查找和限定名称查找，以及在需要时的参数依赖查找和模板参数推导：

* 非限定名称查找([Unqualified name lookup](https://en.cppreference.com/w/cpp/language/unqualified_lookup))：当使用未限定的名称时（如std），编译器会在全局或命名空间作用域内查找该名称的声明。对于函数名来说，非限定名称查找还包括参数依赖查找。
* 限定名称查找([Qualified name lookup](https://en.cppreference.com/w/cpp/language/qualified_lookup))：当名称前有明确的命名空间或作用域限定符时（如std::cout），编译器会在指定的命名空间或作用域内查找。
* 参数依赖查找（ADL）：在函数调用时，如果函数名称未限定，编译器还会在函数参数类型的命名空间中查找可能的函数声明。
* 重载解析：如果名称查找找到了多个具有相同名称的声明，编译器将根据上下文和参数类型来选择最合适的声明。

对于函数和函数模板名称，名称查找可以将多个声明与同一名称关联起来，并且可能从参数依赖查找中获得额外的声明（模板参数推导也可能适用），这一组声明集被传递给重载解析，来选择最终要使用的声明。完成选择之后，才会考虑成员访问规则，即其仅在名称查找和重载解析之后考虑。

对于所有其他名称（变量、命名空间、类等），名称查找只能将多个声明关联到同一个实体，否则它必须产生单一声明，以便程序能够编译。在作用域中查找名称时，会找到该名称的所有声明，有一个例外，被称为“struct hack”或“类型/非类型隐藏。

> 什么是 struct hack
>
> 同一作用域内的名称冲突：在C++中，如果在同一作用域内，一个名称被用作不同类型的声明，比如一部分声明是类型（如类、结构体、联合体或枚举），而另一部分声明是非类型（如变量、非静态数据成员或枚举器），这时会发生名称冲突。
>
> 当名称冲突发生时，如果类型名称（类、结构体、联合体或枚举）不是通过typedef声明的，那么这个类型名称在查找时会被隐藏。这意味着，当你尝试使用这个名称时，编译器会首先查找非类型名称。
>
> 尽管发生了名称冲突，但C++编译器不会报错，因为这种隐藏是有意为之的，以允许类型和非类型名称共存于同一作用域。

```c
// 要访问被隐藏的类型名称，你必须使用详细类型说明符（elaborated type specifier）。这通常涉及到使用作用域运算符::来指定完整的类型名称。例如，如果你有一个名为MyType的类和同名的变量MyType，你可以使用::MyType来指代类类型
class MyType {};

int MyType = 10;  // 同一个作用域内，MyType作为变量名

// 访问类类型，需要使用作用域运算符
MyType::MyType instance;  // 正确，访问类MyType
```

## [非限定名称查找](https://en.cppreference.com/w/cpp/language/unqualified_lookup)

非限定名称查找是指在名字没有出现在域运算符`::`右边的情况下，对名称进行查找的过程。查找会在多个作用域中进行，直到找到至少一个声明为止：

* 文件作用域：在全局（顶层命名空间）中，查找会在名称使用之前的作用域中进行。
* 命名空间作用域：如果在用户声明的命名空间中使用名称，首先会搜索该命名空间，然后是包含该命名空间的外部命名空间，依此类推，直到达到全局命名空间。
* 类定义：在类定义中的任何位置使用名称时，会搜索类定义本身、其基类、嵌套类的定义等
  * 类体内查找：如果在类定义中使用了一个名称，首先会在该类的定义范围内查找，直到使用该名称的位置。
  * 基类查找：如果在当前类中没有找到名称，查找会继续到当前类的直接基类定义中。如果基类中也没有找到，并且基类还有自己的基类，查找会递归地继续到更深层次的基类中。
  * 嵌套类查找：如果当前类是嵌套在另一个类中的，查找会扩展到包含这个嵌套类的外部类的定义中。同时，也会查找外部类的所有基类。
  * 局部类查找：如果类是局部的（即在函数或代码块内定义的），或者嵌套在另一个局部类中，查找会在定义该类的代码块范围内进行，直到类的定义点。
  * 命名空间查找：如果类是命名空间的成员，或者嵌套在命名空间成员类中，或者类是命名空间中函数的局部类，查找会在包含该类的命名空间的作用域内进行。如果需要，查找会继续到包含该命名空间的外部命名空间，直到达到全局作用域。

在查找时，还存在一些特殊的规则，以下仅举两例：

* 比如在查找域运算符`::`左边的名字时，会忽略函数、变量、枚举等，只有类型名称会被查找
* 在类内部声明的友元函数，其名称查找规则与成员函数相同。在类外部定义的友元函数，其查找规则与命名空间中的函数相同。

## [限定名称查找](https://en.cppreference.com/w/cpp/language/qualified_lookup)

限定名称查找用于处理在作用域解析操作符::右侧出现的名称。这种名称可以指向：

* 类成员（包括静态和非静态函数、类型、模板等）
* 命名空间成员（包括另一个命名空间）
  * 通常在命名空间的作用域查找。特例是对模版参数中的名字，会在当前作用域查找，而不是模版名称的作用域

    ```c
    namespace N
    {
        template<typename T>
        struct foo {};
 
        struct X {};
    }
 
    N::foo<X> x; // Error: X is looked up as ::X, not as N::X
    ```

* 枚举
  * 如果左侧名称查找结果是一个枚举（无论是限定的还是非限定的），右侧名称查找必须是该枚举中的一个枚举器，否则程序是不正确的

如果::左侧没有任何内容，查找只考虑在全局命名空间范围内的声明（或者通过using声明引入到全局命名空间的声明）。这允许引用被局部声明隐藏的名称。

在对::右侧的名称进行查找之前，必须先完成对左侧名称的查找。查找可能是限定的或非限定的，取决于该名称左侧是否有另一个::。查找仅考虑命名空间、类类型、枚举和模板特化（它们是类型）。

如果左侧找到的名称不是指一个命名空间或类、枚举或依赖类型，程序是不正确的（ill-formed）。

当限定名称用作声明时，对跟随该限定名称的同一声明中使用的名称进行非限定查找，但不对前置名称进行查找。查找在成员的类或命名空间的作用域内执行:

```c
class X {};
 
constexpr int number = 100;
 
struct C
{
    class X {};
    static const int number = 50;
    static X arr[number];
};
 
X C::arr[number], brr[number];    // Error: look up for X finds ::X, not C::X
C::X C::arr[number], brr[number]; // OK: size of arr is 50, size of brr is 100
```

## [参数依赖查找](https://en.cppreference.com/w/cpp/language/adl)

Argument-dependent lookup (ADL) 是一组规则，用于在函数调用表达式中查找未限定的函数名称，包括对重载运算符的隐式函数调用。除了通常的未限定名称查找所考虑的作用域和命名空间外，这些函数名称还会在其参数的命名空间中进行查找。

```c
#include <iostream>
 
int main()
{
    std::cout << "Test\n"; // There is no operator<< in global namespace, but ADL
                           // examines std namespace because the left argument is in
                           // std and finds std::operator<<(std::ostream&, const char*)
    operator<<(std::cout, "Test\n"); // Same, using function call notation
 
    // However,
    std::cout << endl; // Error: “endl” is not declared in this namespace.
                       // This is not a function call to endl(), so ADL does not apply
 
    endl(std::cout); // OK: this is a function call: ADL examines std namespace
                     // because the argument of endl is in std, and finds std::endl
 
    (endl)(std::cout); // Error: “endl” is not declared in this namespace.
                       // The sub-expression (endl) is not an unqualified-id
}
```

ADL 的工作原理可以总结为以下步骤：

* 首先会判断是否执行ADL：如果通常的未限定查找结果中包含类成员声明、块作用域中的函数声明（非using声明）或任何非函数或函数模板的声明，则不执行ADL。
* 然后对每个参数进行类型检查：对于函数调用表达式中的每个参数，会检查其类型以确定将添加到查找中的相关命名空间和类（具体不同类型对应的命名空间规则比较复杂，详见cppreference）
* 接着关联集合：基于参数类型，会构建一个关联的命名空间和类的集合。例如，对于类类型参数，包括该类本身、其所有直接和间接基类以及这些类最内层的包围命名空间。
* 查找合并：将普通未限定查找找到的声明集合与ADL找到的声明集合合并，并应用特殊规则，例如，通过ADL可见的关联类中的友元函数和函数模板，即使它们在普通查找中不可见。

ADL 使得在类同名空间中定义的非成员函数和运算符，如果通过ADL被找到，则被视为该类公共接口的一部分：

```c
template<typename T>
struct number
{
    number(int);
    friend number gcd(number x, number y) { return 0; }; // Definition within
                                                         // a class template
};
 
// Unless a matching declaration is provided gcd is
// an invisible (except through ADL) member of this namespace
void g()
{
    number<double> a(3), b(4);
    a = gcd(a, b); // Finds gcd because number<double> is an associated class,
                   // making gcd visible in its namespace (global scope)
//  b = gcd(3, 4); // Error; gcd is not visible
}
```

如果函数调用是模板函数，并且模板参数是显式指定的，那么必须通过普通查找找到模板的声明。如果没有找到声明，就会遇到一个语法错误，因为编译器会期望一个已知的名称后面跟一个小于号（'<'）：

```c
namespace N1
{
    struct S {};
 
    template<int X>
    void f(S);
}
 
namespace N2
{
    template<class T>
    void f(T t);
}
 
void g(N1::S s)
{
    f<3>(s);     // Syntax error until C++20 (unqualified lookup finds no f)
    N1::f<3>(s); // OK, qualified lookup finds the template 'f'
    N2::f<3>(s); // Error: N2::f does not take a non-type parameter
                 //        N1::f is not looked up because ADL only works
                 //              with unqualified names
 
    using N2::f;
    f<3>(s); // OK: Unqualified lookup now finds N2::f
             //     then ADL kicks in because this name is unqualified
             //     and finds N1::f
}
```

另一个加深理解的例子：

```c
namespace A
{
    struct X;
    struct Y;
 
    void f(int);
    void g(X);
}
 
namespace B
{
    void f(int i)
    {
        f(i); // Calls B::f (endless recursion)
    }
 
    void g(A::X x)
    {
        g(x); // Error: ambiguous between B::g (ordinary lookup)
              //        and A::g (argument-dependent lookup)
    }
 
    void h(A::Y y)
    {
        h(y); // Calls B::h (endless recursion): ADL examines the A namespace
              // but finds no A::h, so only B::h from ordinary lookup is used
    }
}
```

## 回到问题

那么回到最开始的问题。

为什么单独编译库的源文件 `xxx.cc` 没有问题呢？ `sort(vec_.begin(), vec_.end(), std::less<double>());`，显而易见，这里虽然没有显式指定`sort`所属的命名空间`std`，但是其参数 `vec_` 和 `less` 是有明确命名空间的，这个命名空间在ADL的过程中被查找，因此最终找到了 `std::sort` 的函数声明。

为什么与 `yyy.h` 一起编译的时候，在没有`include`的情况下也会失败呢？ 是因为在全局查找的过程中首先找到了 `namespace sort`，所以此时编译器指出，`sort(vec_.begin(), vec_.end(), std::less<double>());` 是有歧义的，编译器不知道哪个是正确的。

## 为什么C++会有ADL

为什么在限定名称查找和非限定名称查找之外，C++还要提供参数依赖查找这样的机制呢？它其实是在规范的查找框架下，提供了一种灵活性的补充：

* 增强的表达能力：ADL允许程序员调用与参数类型相关的非成员函数，而不必显式地指定这些函数所在的命名空间。这提高了代码的可读性和表达能力。
* 支持泛型编程：在模板编程中，ADL使得模板能够使用与模板参数类型相关的特定操作，而无需程序员显式地指定这些操作的命名空间。这使得模板更加通用和灵活。
* 避免命名冲突：ADL通过在参数类型的命名空间中查找函数，减少了全局命名空间的污染，有助于避免命名冲突。
* 支持自定义操作：ADL使得程序员可以在自己的类型所在的命名空间中定义与标准库类型相关的操作，如自定义的swap函数。这样，当使用标准库算法时，这些自定义操作可以被自动使用。
* 符合C++的设计哲学：C++语言的设计哲学之一是提供强大而灵活的工具，以支持各种编程范式。ADL是这一哲学的体现，它提供了一种自然而直观的方式来处理与类型相关的操作。
* 历史原因：ADL是C++早期版本中就已经存在的特性，它随着语言的发展而逐渐演化，成为C++中不可或缺的一部分。

## 参考引用

> 关于"在C++中确定一个名称"这一相关话题，本文仍有一些未提及的场景，比如模板参数推导、重载解析等，可以参考：

- [Name lookup](https://en.cppreference.com/w/cpp/language/lookup)
- [Template argument deduction](https://en.cppreference.com/w/cpp/language/function_template)
- [Overload resolution](https://en.cppreference.com/w/cpp/language/overload_resolution)
