# Specialization & Overloading

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