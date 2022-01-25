# 模板实战

## 显式实例化

显式实例化，由`template`和实例化实体的声明组成，该声明式一个已经用实参完全替换参数之后的声明。

```cpp
// 基于`double`实例化`print_typeof()`
template void print_typof<double>(double const&);

// 基于`int`显式实例化`MyClass<>`的构造函数
template MyCalss<int>::MyClass();

// 基于`int`显式实例化函数模板`max()`
template int const& max(int const&, int const&);
```

还可以显式实例化类模板，这样会同时实例化它的所有成员。需要注意的是：类成员只能实例化一次。

```c++
template class Stack<int>;

template Stack<std::string>::Stack();

template Stack<int>::Stack();       // Error: 已经显式实例化过int类型，不能重复实例化

```

## 模板的分离模型

在一个文件里面定义模板，并在模板的定义和声明前面加上关键字export。

## 调试模板

concept表示：在模板库中重复需求的约束集合

调试模板代码的主要工作是判断模板实现和模板定义中哪些concept被违反了。

## 小结

* 模板会给原始的”编译器+链接器“模型带来挑战，因此，需要其他方式组织代码：包含模型、显式实例化、分离模型
* 大多数情况下，建议使用包含模型（模板代码都放在头文件）
* 通过把模板声明和模板定义放在不同的头文件，可以很容易在包含模板和显示实例化之间做出选择
* 分离模型使用`export`关键字
* 测试模板代码具有挑战性
* 模板实例化会生成很长的名称
* 为了充分利用预编译代码，最好使`#include`的顺序相同
  