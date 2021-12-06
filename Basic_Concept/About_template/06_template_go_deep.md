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