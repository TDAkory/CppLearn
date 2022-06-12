# Specialization & Overloading

- [Specialization & Overloading](#specialization--overloading)
  - [When "Generic Code" Doesn't Quite Cut It](#when-generic-code-doesnt-quite-cut-it)
    - [透明自定义](#透明自定义)
  - [重载函数模板](#重载函数模板)
    - [签名](#签名)
    - [重载的函数模板的局部排序](#重载的函数模板的局部排序)
    - [排序原则](#排序原则)
  - [显式特化](#显式特化)
    - [全局的类模板特化](#全局的类模板特化)
    - [全局函数模板特化](#全局函数模板特化)
    - [全局成员特化](#全局成员特化)
  - [局部的类模板特化](#局部的类模板特化)

## When "Generic Code" Doesn't Quite Cut It

```cpp
template<typename T> 
class Array { 
  private: 
    T* data; 
    ...  
  public: 
    Array(Array<T> const&); 
    Array<T>& operator = (Array<T> const&); 
    void exchange_with (Array<T>* b) {  // cheaper
        T* tmp = data; 
        data = b->data; 
        b->data = tmp; 
    } 
    T& operator[] (size_t k) { 
        return data[k]; 
    } 
    ...  
}; 
template<typename T> inline 
void exchange (T* a, T* b)  // expensive
{ 
     T tmp(*a); 
     *a = *b; 
     *b = tmp; 
}
```

### 透明自定义

```cpp
template<typename T> inline 
void quick_exchange(T* a, T* b)                // (1)  
{ 
    T tmp(*a); 
    *a = *b; 
    *b = tmp; 
} 
template<typename T> inline 
void quick_exchange(Array<T>* a, Array<T>* b)  // (2)  
{ 
    a->exchange_with(b); 
} 
void demo(Array<int>* p1, Array<int>* p2) 
{ 
    int x, y; 
    quick_exchange(&x, &y);                    // uses (1)  
    quick_exchange(p1, p2);                    // uses (2)  
} 
```

在其他条件一样的情况下，重载解析规则会优先选择更加特殊的模板。

## 重载函数模板

两个同名的函数模板可以同时存在，可以对它们进行实例化，使它们具有相同的参数类型

```cpp
// funcoverload.h
template<typename T> 
int f(T) 
{ 
    return 1; 
} 
template<typename T> 
int f(T*) 
{ 
    return 2; 
} 
```

```cpp
#include <iostream> 
#include "funcoverload.h" 
int main() 
{ 
    std::cout << f<int*>((int*)0) << std::endl;     // 1
    std::cout << f<int>((int*)0)  << std::endl;     // 2
}
```

详细分析`f<int *>((int *)0)`的匹配过程：语法`f<int *>`说明我们希望用`int *`来替换第一个模板参数，并且这种显式替换不依赖模板参数推导。该例子中，有两个f模板，生成的重载函数集包括：`f<int *>(int *)`、`f<int *>(int **)`，然而，调用实参`(int *)0`的类型是`int *`，因此会和第一个模板生成的函数更好的匹配。

### 签名

**只要具有不同的签名，两个函数就可以在同一个程序中同时存在。**对函数的签名定义如下：

* 非受限函数的名称（或者产生自函数模板的这类名称）
* 函数名称所属的类作用域或者名字空间作用域：如果函数名称是具有内部连接的，还包括该名称声明所在的翻译单元
* 函数的const、volatile后者const volatile限定符（前提是它是一个具有这类限定符的成员函数）
* 函数参数的类型（若该函数产生自模板，指模板参数被替换之前的类型）
* 如果这个函数是产生自函数模板，则包括他的返回类型
* 如果这个函数是产生自函数模板，则包括模板参数和模板实参

```cpp
// 如下模板和实例化实体可以在同一个程序中同时存在
template<typename T1, typename T2> 
void f1(T1, T2); 

template<typename T1, typename T2> 
void f1(T2, T1); 

template<typename T> 
long f2(T); 

template<typename T> 
char f2(T);
```

但是，实例化过程仍然可能导致重载二义性。这种情况下，这类模板需要存在于不同的翻译单元。

### 重载的函数模板的局部排序

```cpp
#include <iostream> 
template<typename T> 
int f(T) 
{ 
    return 1; 
} 
template<typename T> 
int f(T*) 
{ 
    return 2; 
} 
int main() 
{ 
    std::cout << f(0) << std::endl;             // 1
    std::cout << f((int*)0)  << std::endl;      // 2
} 
```

考虑第一个调用：实参类型是int，可以个第一个模板匹配。无法匹配第二个模板因为其参数类型总是一个指针。在这里，重载解析没有发挥作用。

考虑第二个调用：两个模板的推导都可以成功，获得两个函数`f<int *>(int *)`、`f<int>(int *)`。按照一般观点，两个函数和实参类型的匹配程度是一样的。这时需要考虑重载解析的额外规则：**选择产生自更特殊的模板的函数**。第二个模板被认为是更加特殊的模板。

### 排序原则

在有多个函数和函数模板名字相同的情况下，一条函数调用语句到底应该被匹配成对哪个函数或哪个模板的调用呢？ C++ 编译器遵循以下先后顺序：

1. 先找参数完全匹配的普通函数（非由模板实例化得到的函数）。
2. 再找参数完全匹配的模板函数。
3. 再找实参经过自动类型转换后能够匹配的普通函数。
4. 如果上面的都找不到，则报错。

## 显式特化

类模板是不可以被重载的，但我们可以选择另一种机制来实现这种透明的自定义类模板的能力，就是显式特化。

类模板和函数模板都是可以被全局特化的。

### 全局的类模板特化

```cpp
template<typename T>
class S {
    void info() { std::cout << "generic (S<T>::info())"; }
};

template<>
class S<void> {
    void msg() { std::cout << "fully specialized (S<void>::msg())"; }   // 全局特化与原来的实现可以毫无关联。只有类模板的名字有关联
};
```

```cpp
// 指定的模板实参列表必须和响应的模板参数列表一一对应
template<typename T>
class Types {
    typedef int I;
};

template<typename T, typename U = typename Types<T>::I>
class S;    // 1

template<>
class S<void> {
    void f();       // 2
};

template<> class S<char, char>;     // 3

template<> class S<char, 0>;    // 错误，不能用0替换U

int main() {
    S<int> *pi;         // 正确，使用1，这里不需要定义
    S<int> e1;          // 错误，使用1，需要定义，但是找不到定义
    S<void> *pv;
    S<void, int> sv;
    S<void, char> e2;
    S<char, char> e3;
}
```

对于特化声明而言，因为它不是模板声明，所以应该使用（位于类外部）的普通成员定义语法，来定义全局类模板特化的成员，不能指定template<>前缀。

```cpp
template<typename T>
class S;

template<> class S<char *> {
    void print() const;
};

void S<char *>::print() const {
    std::cout << "no template<> ahead" << std::endl;
}
```

可以用全局模板特化来代替对应的泛型模板的某个实例化体。但两者不能共存在同一个程序中，会导致编译期错误。

```cpp
template<typename T>
class Invalid {};

Invalid<double> x1; // 产生一个实例化

template<>
class Invalid<double>;  // 错误：Invalid<double>已经被实例化了
```

上述错误如果发生在不同的编译单元，则错误很难被捕捉。

### 全局函数模板特化

从语法和原则上讲，全局函数模板特化和类模板特化基本是一致的。唯一的区别在于，函数模板特化引入了重载和实参演绎这两个概念。

借助实参演绎的情况下，全局特化可以不声明显式的模板实参。

```cpp
template<typename T>
int f(T) // (1)
{
    return 1;
}

template<typename T>
int f(T*) // (2)
{
    return 2;
}

template<> int f(int) // OK: specialization of (1)
{
    return 3;
}

template<> int f(int*) // OK: specialization of (2)
{
    return 4;
}
```

全局函数模板特化不能包含缺省的实参值，但是，对于基本模板所指定的任何缺省实参，显式特化版本都可以应用这些缺省实参

```cpp
template<typename T>
int f(T, T x = 42) {
    return x;
}

template<> int f(int, int = 35) // ERROR!
{
    return 0;
}

template<typename T>
int g(T, T x = 42)
{
    return x;
}

template<> int g(int, int y)
{
    return y/2;
}

int main()
{
std::cout << g(0) << std::endl; // should print 21
}
```

### 全局成员特化

除了成员模板外，类模板的成员函数和静态成员变量也可以被全局特化。

```cpp
template<typename T>
class Outer { // (1)
  public:
    template<typename U>
    class Inner { // (2)
      private:
        static int count; // (3)
    };
    static int code; // (4) 
    void print() const { // (5)
        std::cout << "generic";
    }
};

template<typename T>
int Outer<T>::code = 6; // (6)

template<typename T> template<typename U>
int Outer<T>::Inner<U>::count = 7; // (7)

template<>
class Outer<bool> { // (8)
  public:
    template<typename U>
    class Inner { // (9)
      private:
        static int count; // (10)
    };
    void print() const { // (11)
    }
};

// 这些定义替代Outer<void>中的定义，类Outer<void>的其他定义默认产生自1处的模板，此外，提供上述声明之后，不能在提供Outer<void>的显式特化
template<>
int Outer<void>::code = 12; // 4的特化

template<>
void Outer<void>::print() const {   // 5的特化
    std::cout << "Outer<void>"; 
}

// 特化成员模板Outer<T>::Inner
template<>
    template<typename X>
    class Outer<wchar_t>::Inner {
      public:
        static long count;  // 成员类型发生了改变  
    }；

template<>
    template<typename X>
    long Outer<wchar_t>::Inner<X>::count;

// 模板Outer<T>::Inner全局特化
template<>
    template<>
    class Outer<char>::Inner<wchar_t> {
      public:
        enum { count = 1};
    };

// 下面的方式是不合法的，template<>不能位于模板实参列表的后面
template<typename X>
template<> class Outer<X>::Inner<void>; // ERROR

// 若外围模板类已经完成了全局特化，Outer<bool>，则它的成员模板也就不存在外围模板了，只需要一个template<>
template<>
class Outer<bool>::Inner<wchar_t> {
  public:
    enum { count = 2};
};
```

## 局部的类模板特化

有时候希望把类模板特化成一个”针对模板实参“的类家族，而不是针对”一个具体实参列表“的全局特化

```cpp
template<typename T>
class List {
    void append(T const &);
    inline size_t length() const;
};

template<>
class List<void *> {
    ...
};

template<typename T>
class List<T *> {
  private:
    // 这里存在的问题是，List<void *>会递归地包含一个相同类型的List<void *>的成员，为了打破递归，可以在局部特化前面先提供一个全局特化
    List<void *> impl;
  public:
    void append(T *p) {
        impl.append(p);
    }

    size_t length() const {
        return impl.length();
    }
}；
```

* 注意：进行匹配时，全局特化优于局部特化。

局部特化声明的参数列表和实参列表，存在如下约束：
1. 局部特化的实参必须和基本模板的响应实参在种类上（类型、非类型、模板）是匹配的
2. 局部特化的参数列表不能有缺省实参，但局部特化可以使用基本类模板的缺省实参
3. 局部特化的非类型实参只能是非类型值，或者是普通的非类型模板参数，而不能是更复杂的依赖型表达式
4. 局部特化的模板实参列表不能和基本模板的参数列表完全等同

```cpp
template<typename T, int I = 3>
class S; // primary template

template<typename T>
class S<int, T>; // ERROR: parameter kind mismatch

template<typename T = int>
class S<T, 10>; // ERROR: no default arguments

template<int I>
class S<int, I*2>; // ERROR: no nontype expressions

template<typename U, int K>
class S<U, K>; // ERROR: no significant difference from primary template
```
