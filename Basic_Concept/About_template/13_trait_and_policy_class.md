# Trait and  Policy

## 例子：累加一个序列

### Fixed traits

```cpp
// traits/accum1.hpp 
#ifndef ACCUM_HPP 
#define ACCUM_HPP 

template <typename T> 
inline 
T accum (T const* beg, T const* end) { 
    T total = T(); // assume T() actually creates a zero value 
    while (beg != end) { 
        total += *beg; 
        ++beg; 
    }
    return total; 
}
#endif // ACCUM_HPP
```

上面的案例，复杂点在于如何正确的为类型生成一个0值，来开始求和过程。在这里使用了`T()`，对于内建数值类型而言，`T()`是符合要求的。

为了引出第一个`trait`模板，考虑如下代码

```cpp
// traits/accum1.cpp 
#include "accum1.hpp" 
#include <iostream> 
int main() {
    // create array of 5 integer values 
    int num[]={1,2,3,4,5}; 
    
    // print average value 
    std::cout << "the average value of the integer values is " 
              << accum(&num[0], &num[5]) / 5
              << '\n'; 
    // create array of character values 
    char name[] = "templates"; 
    int length = sizeof(name)-1;
    
     // (try to) print average character value 
     std::cout << "the average value of the characters in \""
               << name << "\" is " 
               << accum(&name[0], &name[length]) / length << '\n';
}
```

```cpp
// EBCDIC机器上输出如下，IBM的字符集，非ASCII
the average value of the integer values is 3 
the average value of the characters in "templates" is -5
```

为了解决上述差异，可以考虑对`assum`调用的每个`T`类型都创建一个关联，所关联的类型就是用来存储累加和的类型，这种关联被看作是类型`T`的一个特征，因此被称为`T`的`trait`。

```cpp
// traits/accumtraits2.hpp 
template<typename T> 
class AccumulationTraits; 

template<> 
class AccumulationTraits<char> { 
    public: typedef int AccT; 
};

template<> 
class AccumulationTraits<short> { 
    public: typedef int AccT; 
};

template<> 
class AccumulationTraits<int> { 
    public: typedef long AccT; 
};

template<> 
class AccumulationTraits<unsigned int> { 
    public: typedef unsigned long AccT; 
};

template<> 
class AccumulationTraits<float> { 
    public: typedef double AccT; 
};
```

改写的accum模板

```cpp
// traits/accum2.hpp 
#ifndef ACCUM_HPP 
#define ACCUM_HPP 

#include "accumtraits2.hpp" 

template <typename T> 
inline 
typename AccumulationTraits<T>::AccT accum (T const* beg, T const* end) { 
    // return type is traits of the element type 
    typedef typename AccumulationTraits<T>::AccT AccT; 
    AccT total = AccT(); // assume T() actually creates a zero value 
    while (beg != end) { 
        total += *beg; ++beg; 
    }
    return total;
}
#endif // ACCUM_HPP
```

### Value Traits

trait可以用来表示：主类型所关联的额外的类型信息。此外，这个额外的信息并不局限于类型，常数和其他类型的值也可以和一个类型进行关联。

解决上面一节程序：默认构造返回0值的问题，可以给AccumulationTraits添加一个value trait

```cpp
template<typename T> 
class AccumulationTraits; 

template<> class AccumulationTraits<char> { 
    public: 
        typedef int AccT; 
        static AccT const zero = 0; };

template<> class AccumulationTraits<short> { 
    public: 
        typedef int AccT; 
        static AccT const zero = 0; };

template<> class AccumulationTraits<int> { 
    public: 
        typedef long AccT; 
        static AccT const zero = 0; };
…
```

新的trait是一个常量，可以在编译器得到结果，accum修改如下：

```cpp
// traits/accum3.hpp #ifndef ACCUM_HPP
#define ACCUM_HPP 
#include "accumtraits3.hpp" 

template <typename T> 
inline 
typename AccumulationTraits<T>::AccT accum (T const* beg, T const* end) { 
    // return type is traits of the element type 
    typedef typename AccumulationTraits<T>::AccT AccT; 
    AccT total = AccumulationTraits<T>::zero; 
    while (beg != end) { 
        total += *beg; ++beg; 
    }
    return total; 
}
#endif // ACCUM_HPP
```

上述方法的缺点是：在所有类的内部，C++只允许我们对整形和枚举类型初始化成静态成员变量。

即便声明和定义分开可以规避这种问题，但该情况下变量的定义对编译器而言是不可知的，编译器不知道位于其他文件的定义。

因此可以做如下修改

```cpp
// traits/accumtraits4.hpp 
template<typename T> 
class AccumulationTraits; 

template<> 
class AccumulationTraits<char> { 
    public: 
        typedef int AccT; 
        static AccT zero() { return 0; }
};

...
```

### Parameterized Tratis 参数化Traits