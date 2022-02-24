# 实例化

- [实例化](#实例化)
  - [On-Demand实例化](#on-demand实例化)
  - [延迟实例化](#延迟实例化)
  - [C++实例化模型](#c实例化模型)
    - [两阶段查找](#两阶段查找)
    - [POI](#poi)
    - [包含模型和分离模型](#包含模型和分离模型)
    - [跨翻译单元查找](#跨翻译单元查找)
    - [一些例子](#一些例子)
  - [几种实现方案](#几种实现方案)
    - [问题](#问题)
    - [贪婪实例化](#贪婪实例化)
    - [询问实例化](#询问实例化)
    - [迭代实例化](#迭代实例化)
  - [显式实例化](#显式实例化)

## On-Demand实例化

当C++编译器遇到模板特化的时候，它会利用所给的实参替换对应的模板参数，从而产生改模板的特化。（On-Demand实例化）

On-Demand实例化表明：**在使用模板（特化）的地方，编译器通常需要访问模板和某些模板常用的整个定义。**

```cpp
template<typename T>class C;    // declaration only

C<int> *p = 0       // fine: definition of C<int> not needed

C<void> *p1 = new C<void>   // wrong: needs the instantiation of class template because the size of C<void> is needed

template<typename T>
class C {
    void f();   // member declaration
};              // class template definition completed

void g(C<int> &c) { // use class template definition
    c.f();          // will need definition of C::f()
}
```

C++的重载解析规则要求：如果候选函数的参数是class类型，则该类型对应的类必须是可见的

```cpp
tamplate<typename T>
class C {
    C(int); // can be implicit conversions 
};

void candidate(C<double> const &);
void candidate(int) {}

int main() {
    candidate(42);  // 前面两个函数声明都可以被调用
}
```

## 延迟实例化

关于模板实例化的程度问题，模糊的回答是：只对确定需要的部分进行实例化。换句话说，编译器会延迟模板的实例化

- 当隐式实例化类模板是，同时也实例化了该模板的每个成员声明，但并没有实例化相应的定义。
  - 若类模板包含了一个匿名的union，则union定义的成员同时也被实例化了
  - 作为实例化模板的结果，虚函数的定义可能 被 或者 没有被 实例化，取决于实现
    - 多数实现会进行实例化，因为**实现虚函数调用机制的内部结构**要求虚函数定义作为链接实体存在
  - 缺省的函数调用实参是否实例化，取决于被调用的函数是否使用了缺省实参；如果不使用缺省实参而是使用了显式实参，则不会实例化

```cpp
template<typename T>
class Danger {
    typedef char Block[N];  // Error for N <=0
};

template <typename T, int N>
class Tricky {
    virtual ~Tricky() {}

    void no_body_here(Safe<T> = 3);
    void inclass() {
        Danger<N> no_boom_yet;
    }

    void error() { Danger<0> boom; }   // 会用0实例化Danger，编译期错误，发生在泛型模板的处理过程
    void unsafe(T (*p)[N]);     // 编译期认为正确，在N没有被替换之前，该声明不会产生错误
    T operator->();
    virtual Safe<T> suspect();
    struct Nested {
        Danger<N> pfew;
    };
    union {
        int align;
        Safe<T> anonymous;
    };
};

int main() {     // Tricky实例化，int 替换 T， 0 替换 N
    Tricky<int, 0> ok;  // 不需要Tricky的所有成员定义，但是缺省构造和析构函数一定会被调用
}

```

- 在main之前，编译器会**基于假设参数出于最理想的情况**，来编译模板定义，检查语法约束和一般的语义约束

## C++实例化模型

### 两阶段查找

对模板进行解析的时候，编译器并不能解析依赖型名称。于是，编译器会在POI(Point of instantiation, 实例化点)再次查找这些依赖型名称

非依赖型名称在首次看到模板的时候就进行查找，因为在第一次查找的时候就可以诊断错误信息

由此得出两阶段查找的概念：**模板解析阶段 & 模板实例化阶段**

### [POI](https://stackoverflow.com/questions/3866215/what-is-poi-and-what-does-it-mean)

通俗理解就是代码中的一个点，在这个点会插入替换后的模板实例

```cpp
class MyInt {
    MyInt(int i);
};

MyInt operator-(MyInt const &);

bool operator>(MyInt const&, MyInt const&);

typedef MyInt Int;

template<typename T>
void f(T i) {
    if (i > 0) {
        g(-i);
    }
}
// 1，函数g不可见
void g(Int) {
    // 2，不能作为POI，不允许模板定义在这里插入
    f<Int>(42); // point fo call，编译器看到这里，知道模板函数f需要用MyInt来实例化，即生成一个POI
    // 3
}
// 4，函数g可见，对于执行非类型特化的引用，C++把它的POI定义在“包含这个引用的定义或声明之后的最近名字空间域”。即4
```

在POI执行第二次查找时，指g(-i)，只是使用了ADL。基本类型int没有关联名字空间，因此如果使用int，就不会发生ADL，也就不能找到函数g，将无法通过编译。

```cpp
// 对于类特化
template<typename T>
class S {
    T m;
};
// 5，对于指向产生自模板的类实例的引用，它的POI定义“包含这个实例引用的定义或声明之前的最近名字空间域”，即5
unsigned long h() {
    // 6
    return (unsigned long)sizeof(S<int>);
    // 7
}
// 8，若POI在这里，sizeof(S<int>)会无效，因为要等编译到8，才能确定S<int>的大小，但是代码就在8之前
```

```cpp
template<typename T>
class S {
    typedef int I;
};

// 1
template<typename T>
void f() {
    S<char>::I car1 = 41;
    typename S<T>::I var2 = 42;
}

int main() {
    f<double>();
}// 2: 2a, 2b
```

`f<double>`的POI在2处，它引用了类特化`S<char>`，雷特和的POI在1处。另外，函数模板还引用了`S<T>`：因为`S<T>`是依赖型的，所以不能像`S<char>`来确定POI

对于非类型实体，二次POI的位置和主POI(`f<double>`)的位置相同；对于类型实体，二次POI的位置位于主POI位置的紧前处。

因此`S<double>`在2a，`f<double>`在2b

在实际应用中，大多数编译器会延迟非内联函数模板的实例化，知道翻译单元末尾才进行真正的实例化。

### 包含模型和分离模型

当遇到POI的时候，编译器要求相应模板的定义必须是可见的。

将模板定义在头文件中，在需要使用到的地方#include这个头文件，属于包含模型

使用export关键字来声明非类型模板，在另一个翻译单元中定义该非类型模板，属于分离模型。

```cpp
// 翻译单元1
#include <iostream>
export template<typename T>
T const &max(T const&, T const&)

int main() {
    std::cout << max(a, b);
}

// 翻译单元2
export template<typename T>
T const &max(T const& a, T const& b) {
    return a < b ? b : a;
}
```

### 跨翻译单元查找

如果将翻译单元1做如下修改

```cpp
// 翻译单元1
#include <iostream>

export template<typename T> T const& max(T const&, T const&);

namespace N {
    class I {
        I(int i) : v(i) {}
        int v;
    };

    bool operator<(I const& a, I const& b) {
        return a.v < b.v;
    }
}

int main() {
    std::cout << max(N::I(7), N::I(42)).v << std::endl; // (3)
}
```

解析模板需要的模板定义和类型I定义分别位于两个编译单元，编译器需要进行两个阶段的查找：

第一阶段，在解析模板（即第一次看到模板定义）的时候，使用普通查找规则和ADL规则对非依赖型名称Inc查找。非受限的依赖型函数名称先使用普通的查找规则进行查找，缓存查找结果，在这个过程中不会进行重载解析。

第二阶段，在POI的时候，使用普通查找规则和ADL来查找依赖型首先名称，依赖型非受限名称仅使用ADL规则查找，然后将其结果与一阶段结果合并组成函数集合，进行重载解析。

### 一些例子

* 包含模型查找问题

```cpp
template<typename T>
void f1(T x) {
    g1(x);  // 1
}

void g1(int) {}

int main() {
    f1(7);  // ERROR: g1 not found
}   // 2. POI for f1<int>(int)
```

当第一次看到模板f1的定义时，编译器注意到非受限名称g1是一个依赖型名称。因此，编译器会在1处使用普通查找规则来查找g1，但该处找不到。

在2处，f1的POI，会在关联名字空间和关联类中再次查找g1，但由于g1的唯一实参是int，而int作为基本类型，没有关联域，因此第2阶段找不到g1。

尽管在f1的POI处可以使用普通查找找到g1，但实际上该例子找不到。

* 分离模型导致跨翻译单元的重载二义性

```cpp
// common.h
export template<typename T>
void f(T);

class A {};
class B {};

class X {
    operator A() { return A(); }
    operator B() { return B(); }
};

// a.cpp
#include "common.h"
void g(A) {}

int main() {
    f<x>(X());
}

// b.cpp
#include "common.h"

void g(B) {}

export template<typename T>
void f(T x) {
    g(X);
}
```

在a.cpp中的f<X>(X())，它解析为文件b.cpp中的导出模板，该模板中的调用g(x)会基于X的类型进行实例化，g会进行两阶段查找：

一次使用普通查找规则在b.cpp中查找，发生在解析模板的时候；会找到g(B)

另一次在文件a.ccp中使用ADL进程查找，在模板实例化的地方；会找到g(A)

借助于自定义的类型转换运算符，两个查找结果都是可行的g函数，因此导致调用二义性

## 几种实现方案

回顾几种主流的C++编译器对包含模型的支持

### 问题

```cpp
// t.h
template<typename T>
class S {
    void f();
};

template<typename T>
void S::f() {}

void helper(S<int> *);

// a.cpp
#include "t.h"

void helper(S<int *s>) {
    s->f();     // 1. first POI
}

// b.cpp
#include "t.h"

int main() {
    S<int> s;
    helper(&s);
    s.f();  // 2. second POI
}
```

如果在两个POI都进行实例化，则会产生二义性问题。因此，编译器从一个翻译单元转移到另一个翻译单元的时候，必须携带某些信息。

相同的问题还会出现在由模板实例化生成的所有可链接实体中，包括：实例化的函数模板、实例化的成员函数模板、实例化的京塔数据成员

### 贪婪实例化

假定链接器知道：特定的实体可以在多个目标文件和程序库中出现，于是，编译器会使用某种方法对这些实体进行标记，当链接器找到多个实例的时候，会保留其中一个

缺点：

* 编译器会在N个实例化实体上浪费时间，最终只保留一个
* 链接器不会检查两个实例化实体是否是一样的
* 与其他解决方案相比，目标文件可能更大，因为相同代码可能会生成多次

优势：

* 保留了源对象之间的元是依赖性‘

### 询问实例化

维护一个数据库，程序中所有翻译单元的编译共享这个数据库。数据库会跟踪：哪些特化已经实例化完毕、这些特化需要依赖哪些源码。当遇到可链接实体的POI时，会根据具体前提操作：
* 不存在所需要的特化：会发生实例化过程，然后生成的特化被放入数据库中
* 所需特化已经存在，但已经过期：再次实例化，并将得到的特化替代数据库中原有的特化
* 一个不需要更新的特化已经存在于数据库中：啥都不干

挑战：
* 需要根据源代码的状态，来正确的维护数据库内容的正确性
* 并行编译的支持

### 迭代实例化

* 不实例化任何所需的可链接特化，直接编译源代码
* 使用预链接器链接目标文件
* 预链接器调用链接器，解析错误信息，确认结果是否缺少某个实例化实体。如果缺少，预链接器调用编译器，编译包含所需模板定义的源代码，然后可选地生成这个缺少的实例化实体
* 重复第三步，直到不再生成新的定义

## 显式实例化

为模板特化显式地生成POI是可行的。由关键字`template`和后面的特化声明组成，所声明的特化就是即将由实例化获得的特化

```cpp
template<typename T>
void f(T) throw T {}

// 4个有效的显式实例化实体
template void f<int>(int) throw(int);   
template void f<>(float) throw(float);
template void f(long) throw(long);  // 模板实参可以通过编译获得
template void f(char);              // 异常规范可以省略，未省略的必须匹配相应的模板
```

```cpp
// 类模板
template<typename T>
class S {
    void f() {]}
};

template void S<int>::f();
template class S<void>;     // 显式实例化类模板特化本身，就同时显式实例化了类模板特化的所有成员
```
