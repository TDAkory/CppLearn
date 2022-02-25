# Template Argument Deduction

## 推导过程

针对一个函数调用，推导过程会比较“调用实参的类型”和“函数模板对应的参数化类型T”，然后针对要被推导的一个或多个参数，分别进行正确的替换。

每个实参-参数对都是独立分析的，如果最后得出的结论存在矛盾，推导过程会失败。

接下来描述如何进行匹配：匹配类型A（实参类型）和参数化类型P（来自参数的声明）。如果被声明的参数是一个引用声明（T&），那么P就是所引用的类型（T），而A仍然是实参类型。否则，P就是所声明的参数类型，A就是实参的类型；如果这个实参的类型是数组或者函数类型，那么还会方式`decay转型`，转化为对应的指针类型，同时还会忽略高层次的`const`和`volatile`限定符。

```cpp
template<typename T> void g(T); // P is T

template<typename T> void g(T&); // P is also T

double x[2];

int const seven = 7;

f(x);       // non reference parameter: T is double *
g(x);       // reference parameter: T is double[20]
f(seven);   // non reference parameter: T is int
g(seven);   // reference parameter: T is int const，在匹配引用参数时，const和volatile限定符是保留的
f(7);       // non reference parameter: T is int
g(7);       // reference parameter: T is int ==> ERROR: can't pass 7 to int&
```

已经知道，**对于引用参数，绑定到该参数的实参是不会进行decay的**。然而，如果遇到字符串类型的实参，缺存在例外：

```cpp
template<typename T>
T const& max(T const& a, T const& b);
```

对于max("Apple", "Pear")，我们可能期望T被推导为`char const *`。然而，两者类型分别是`char const[6]`和`char const[5]`，不存在数组到指针的decay转型，因此会产生解析错误。

## 推导的上下文

对于比T复杂很多的参数化类型，也可以与给定的实参进行匹配

```cpp
template<typename T>
void f1(T*)

template<typename E, int N>
void f2(E(&)[N]);

template<typename T1, typename T2, typename T3>
void f3(T1 (T2::*)(T3*));

class S {
    void f(double *);
};

void g(int ***ppp) {
    bool b[42];
    f1(ppp);        // deduces T to be int**
    f2(b);          // deduce E to be bool & N to be 42
    f3(&S::f);      // deduce T1 = void, T2 = S, T3 = double
}
```

负载的类型声明都是产生自基本的构造（指针、引用、数组、函数声明、成员指针声明、template-id等）；匹配的过程就是从最顶层的构造开始，不断递归各种组成元素。这些构造也被成为腿套上下文。

然而，有些构造不能作为推导上下文：

* 受限的类型名称。`Q<T>::X`的类型不能被用来推导模板参数`T`
* 除了非类型参数，模板参数还包含其他成分的非类型表达式。`S<I+1>`不能被用来推导`I`，也不能通过匹配诸如`ing(&)[sizeof(S<T>)]`类型来推导`T`


## 特殊的推导情况

1. 取函数模板地址的时候，P是函数模板声明子的参数化类型，A是被赋值的指针

```cpp
template<typename T>
void f(T, T);

void (*pf)(char, char) = &f;    // P void(T, T) ; A void(char, char)
```

char替换T，推导成功。pf被初始化为“特化f<char>”的地址

2. 转型运算符模板

```cpp
class S {
    template<typename T, int N>operator T[N]&();    
};

void f(int (&)[20]);

void g(S s) {
    f(s);       // S转型为int(&)[20]， A是int[20], P是T[N], 于是int替换T，20替换N，编译成功
}
```

## 可接受的实参转型