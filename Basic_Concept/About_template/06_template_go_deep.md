# 深入模板基础

## 参数化声明

```cpp
template <typename T>
class List {            // 外部作用域的类模板
    public:
        template <typename T2> List(List<T2> const &);  // 成员函数模板
};

template <typename T>
    template <typename T2>
List<T>::List(List<T2> const &b) {}     // 位于类外部的成员函数模板定义

template <typename T>
int Length(List<T> const&);         // 外部作用域的函数模板

class Collection {
    template <typename T>
    class Node {            // 类内部的类模板定义

    };

    template <typename T>
    class Handle;           // 类内部的类模板声明

    template <typename T> T* alloc() {}     // 类内部的成员函数模板定义，显式内联
};

template <typename T>
class Collection::Handle {      // 在类外部定义的成员类模板

};
```

在所属外围类的外部定义的成员模板可以具有多个模板参数，由外向内依次指明模板的实参。

联合模板也是允许的，被看为类模板的一种

```cpp
template <typename T>
union AllocChunk {
    T object;
    unsigned char bytes(sizeof(T));
};
```

和函数声明一样，函数模板声明也可以具有缺省调用实参，并且缺省实参的参数也可以依赖模板参数

```cpp
template<typename T>
void report_top(Stack<T> const&, int number = 10);

template<typename T>
void fill(Array<T> *, T const & = T());     // T() is zero for built-in types
```

显然，当fill函数被调用时，如果提供了第2个函数调用参数的话，就不会实例化这个确实实参。即：即使不能基于特定类型T来实例化缺省调用实参，可能也不会出现错误。如：

```
class Value {
    public:
        Value(int);
};

void init(Array<Value> *array) {
    Value zero(0);

    fill(array, zero);  // right, 没有使用T()
    fill(array);        // wrong, 使用了T(), 但是当T = Value，没有定义缺省构造函数
}
```

除了两种基本类型的模板，还可以使用相似的符号来参数化如下三种声明：

- 类模板的成员函数
- 类模板的嵌套类成员
- 类模板的静态数据成员

它们的定义都不是自身模板（first-class template），而是依赖外围类模板参数。 

## 虚函数

成员函数模板不能为声明为虚函数，因为成员函数模板的实例化个数是运行期确定的，而虚函数表需要编译器确定，因此产生了冲突。

## 模板的链接

模板名字在其作用域下必须是唯一的，除非函数模板可以被重载。另外，类模板不能和另外一个实体共享名字。

模板通常具有默认的链接属性，但不能具有C链接属性，可以存着非标准（依赖编译器实现）的链接属性。

```cpp
extern "C++" template <typenamt T>
void normal();                          // OK, 缺省情况，链接规范可以不写

extern "C" template <typename T>
void invalid();                         // wrong, 模板不能具有C链接

extern "Xroma" template <typename T>
void xroma_link();                      // 非标，OK
```

## 模板参数

- 类型参数：通过`typename`或`class`引入
- 非类型参数：在编译期或链接期可以确定的常量值（整形或枚举、指针、引用）；只能是右值，不能被取地址或赋值
- 模板的模板参数：代表类模板的占位符，声明和类模板很类似，但不可以使用关键字`struct`和`union`

```cpp
tempalte<typename, int>
class X;            // 在不引用模板参数的情况下，可以省略不写

template<typename T, T* Root, template<T*> class Buf>
class Structure;    // 后面的模板参数可以引用前面的模板参数，反之则不行
```

```cpp
// 模板的模板参数的参数，如A，可以具有缺省模板参数
template <tempalte <typename T,
                    typename A = MyAllocator> class Container>
class Adaptation {
    Container<int> storage;     // 等同于 Container<int, MyAllocator>
};

// 模板的模板参数，其参数名称只可以被自身的其他参数声明使用
template <template<typename T, T*> class Buf>
class Lexer {
    static char storage[5];
    Buf<char, &Lexer<Buf>::storage[0]> buf;
};

template<template<typename T> class List>
class Node {
    static T *storage;      // ERROR: 模板的模板参数的参数在这里不能被使用
};
```

## 缺省模板参数

- 只有类声明可以具有缺省模板实参
- 与缺省的函数参数类似：对于模板参数，只有在其之后的模板参数都提供了缺省实参的情况下，该模板参数才可以声明为缺省模板参数
- 缺省实参不能重复声明

```cpp
template <typename T = void>
class Value;

template <typename T = void>
class Value;            // ERROR
```

## 模板实参

在实例化模板时，用来替换模板参数的值，可以通过以下几种方式确定：

- 显式模板实参
- 注入式类名称
- 缺省模板实参
- 实参演绎（推导）


### 函数模板实参

```cpp
template <typename T>
inline T const & max (T const &a, T const &b) {
    return a < b ? b : a;
}

int main() {
    max<double>(1.0, -3.0); // 显式模板实参
    max(1.0, -2.0);         // 隐式推断为double
    max<int>(1.0, 3.0);     // 显式int，禁止推导
}
```

对于无法被演绎的参数，最好把这些参数放在模板参数列表的开始处，从而可以显式指定这些参数。

```cpp
template <typenamt Dst, typename Src>
inlien Dst implict_cast(Src const &x) {}    // Src可以被推导，Des无法被推导
```

### 类型实参

- 局部类和局部枚举，不可以作为模板的类型实参
- 未命名的class类型或者未命名的枚举类型不能作为模板的实参

```cpp
template <typename T> class List {};

typedef struct {
    double x, y, z;
} Point;

typedef enum {red, green, blue} *ColorPtr;

int main() {
    struct Association {
        int *p;
        int *q;
    };

    List<Association *>error1;  // error: 不可以应用局部类型
    List<ColorPtr> error2;      // error: 不可以使用未命名的量
    List<Point> ok;             // ok
}
```

### 非类型实参

非类型模板实参是哪些替换非类型参数的值，如下：

- 某一个具有正确类型的非类型模板参数
- 一个编译期整型常量。只有在参数类型和值类型能够匹配，或者可以隐式转换的前提洗按合法
- 有单目已婚算法&的外部变量或者函数地址
- 对于引用类型的非类型模板参数，没有&运算符的外部变量和外部函数
- 指向成员的指针常量，类似`&C::m`

当实参匹配“指针或引用”的时候，用户定义的类型转换或派生类到基类的类型转换都不会被考虑。即使在其他环境下，这些转换有效，在模板推导中都是无效的。

隐式转换的唯一应用：给实参加上`const` `volatile`

```cpp
template <typenamt T, T nontype_param>
class C;

C<int, 33> *c1;     // integer type

int a;
C<int *,&a> *c2;    // address of an external variable

void f();
void f(int);
C<void (*)(int), &f> *c3;   // name of a function

class X {
    int n;
    static bool b;
};

C<bool&, X::b> *c4;         // static class members are acceptable variable
C<int X::*, &X::n> *c5;     // example of a pointer-to-member constant

template <typename T>
void func();

C<void(), &func<double>> *c6;   // function template instantiations are functions too
```

模板实参的一个普遍约束是：**程序构建时，编译器或链接器可以确定实参的值**

一些常值不能作为有效的非类型参数：

- 空指针
- 浮点型数值
- 字符串

```cpp
// 一些错误例子
template <typename T, T nontype_param>
class C;

class Base {
    int i;
} base;

class Derived : public Base {} derived;

C<Base *, &derived> *err1;      // 错误：不考虑派生类到基类的转换

C<int & base.i> *err2;          // 错误：域运算符.后面的变量不会被看成变量

int a[10];
C<int *, &a[0]> *err3;          // 错误：单一数组元素的地址不可取
```

### 模板的模板参数