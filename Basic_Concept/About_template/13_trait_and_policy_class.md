# Trait and  Policy

- [Trait and  Policy](#trait-and--policy)
  - [例子：累加一个序列](#例子累加一个序列)
    - [Fixed traits](#fixed-traits)
    - [Value Traits](#value-traits)
    - [Parameterized Tratis 参数化Traits](#parameterized-tratis-参数化traits)
    - [`policy`和`policy`类](#policy和policy类)
    - [`trait` 和 `policy` 的区别](#trait-和-policy-的区别)
    - [成员模板和模板的模板参数](#成员模板和模板的模板参数)
    - [运用普通的迭代器进行累积](#运用普通的迭代器进行累积)
  - [类型函数](#类型函数)
    - [确定元素的类型](#确定元素的类型)

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

参数化trait主要的目的在于：添加一个具有缺省值的模板参数，而且改缺省值由前面介绍的trait模板决定。

```cpp
// traits/accum5.hpp 
#ifndef ACCUM_HPP 
#define ACCUM_HPP 

#include "accumtraits4.hpp" 

template <typename T, typename AT = AccumulationTraits<T>> 
class Accum { 
  public: 
    static typename AT::AccT accum (T const* beg, T const* end) { 
        typename AT::AccT total = AT::zero(); 
        while (beg != end) { 
            total += *beg; 
            ++beg; 
            }
        return total; 
    }
};

template <typename T> 
inline 
typename AccumulationTraits<T>::AccT accum (T const* beg, T const* end) { 
    return Accum<T>::accum(beg, end); 
}

template <typename Traits, typename T> 
inline 
typename Traits::AccT accum (T const* beg, T const* end) { 
    return Accum<T, Traits>::accum(beg, end); 
}
#endif // ACCUM_HPP
```

### `policy`和`policy`类

除了数值的求和，求和语义还能应用在其他场景，比如对序列中给定的值求和，对字符串进行凭借，甚至于在序列中找到一个最大值。这些情况中，针对`accum`的操作，唯一需要改变的就是`total+=*beg`这一句，于是，我们把这个操作称为该累加过程的一个`policy`。因此，一个`policy`类就是一个提供了一个接口的类，该接口能够在算法中应用一个或多个`policy`。

```cpp
// traits/accum6.hpp 
#ifndef ACCUM_HPP 
#define ACCUM_HPP 

#include "accumtraits4.hpp" 
#include "sumpolicy1.hpp" 

template <typename T, typename Policy = SumPolicy, typename Traits = AccumulationTraits<T> > class Accum { 
  public: 
    typedef typename Traits::AccT AccT; 
    static AccT accum (T const* beg, T const* end) { 
        AccT total = Traits::zero(); 
        while (beg != end) { 
            Policy::accumulate(total, *beg); 
            ++beg; }
        return total; 
    } 
};
#endif // ACCUM_HPP
```

```cpp
// traits/sumpolicy1.hpp 
#ifndef SUMPOLICY_HPP 
#define SUMPOLICY_HPP 

class SumPolicy { 
  public: 
    template<typename T1, typename T2> 
    static void accumulate (T1& total, T2 const & value) { 
        total += value; 
    }
};
#endif // SUMPOLICY_HPP
```

考虑如下程序，试图计算几个值的乘积

```cpp
// traits/accum7.cpp 
#include "accum6.hpp" 
#include <iostream> 

class MultPolicy { 
  public:
    template<typename T1, typename T2> 
    static void accumulate (T1& total, T2 const& value) { 
        total *= value; 
    } 
};

int main() { 
    // create array of 5 integer values 
    int num[]={1,2,3,4,5}; // print product of all values 
    std::cout << "the product of the integer values is " 
              << Accum<int,MultPolicy>::accum(&num[0], &num[5]) 
              << '\n'; 
}

// the product of the integer values is 0
```

问题在于，对于乘法，0不是一个合理的初值。这提醒我们：**不同的trait和不同的policy应该是互相交互的，我们应该更加细心的进行模板设计。**

### `trait` 和 `policy` 的区别

* `trait`：用来刻画一个事物的特性
* `policy`：为了某种有益或有利的目的而采用的一系列动作

### 成员模板和模板的模板参数

可以把SumPolicy改写成一个模板：

```cpp
// traits/sumpolicy2.hpp 
#ifndef SUMPOLICY_HPP 
#define SUMPOLICY_HPP 

template <typename T1, typename T2> 
class SumPolicy { 
 public: 
    static void accumulate (T1& total, T2 const & value) { 
        total += value; 
    }
};
#endif // SUMPOLICY_HPP
```

```cpp
// traits/accum8.hpp 
#ifndef ACCUM_HPP 
#define ACCUM_HPP 

#include "accumtraits4.hpp" 
#include "sumpolicy2.hpp" 

template <typename T, 
          template<typename,typename> class Policy = SumPolicy, 
          typename Traits = AccumulationTraits<T> > 
class Accum { 
  public: 
    typedef typename Traits::AccT AccT; 
    static AccT accum (T const* beg, T const* end) { 
        AccT total = Traits::zero();
        while (beg != end) { 
            Policy<AccT,T>::accumulate(total, *beg);
            ++beg; 
            }
        return total; 
    } 
};
#endif // ACCUM_HPP
```

通过模板的模板参数访问policy class的优点在于：借助于某个依赖于模板参数的类型，可以很容易地让policy class携带一些状态信息。
缺点在于：policy必须被实现为模板，且在接口中还定义了模板参数的确切个数，对扩展不是很友好。

如果使用成员模板的方式，可以做到如下扩展：

```cpp
// traits/sumpolicy3.hpp
#ifndef SUMPOLICY_HPP 
#define SUMPOLICY_HPP 
template<bool use_compound_op = true> 
class SumPolicy { 
  public:
    template<typename T1, typename T2> 
    static void accumulate (T1& total, T2 const & value) { 
        total += value; 
    } 
};

template<> class SumPolicy<false> { 
  public:
    template<typename T1, typename T2> 
    static void accumulate (T1& total, T2 const & value) { 
        total = total + value;
    } 
};
#endif // SUMPOLICY_HPP
```

### 运用普通的迭代器进行累积

## 类型函数

* 值函数：参数和返回结果都是值
* 类型函数：接收某些类型实参，并且生成一个类型作为函数的返回结果。比如`sizeof`

类模板也可以作为类型函数。类型函数的参数可以是模板参数，结果是抽取出来的成员类型或成员常量。比如，将sizeof运算符改变成如下接口：

```cpp
// traits/sizeof.cpp 
#include <stddef.h> 
#include <iostream> 

template <typename T> 
class TypeSize { 
    public: static size_t const value = sizeof(T); 
};

int main() { 
    std::cout << "TypeSize<int>::value = " << 
              TypeSize<int>::value 
              << std::endl; 
}
```

### 确定元素的类型

```cpp
// traits/elementtype.cpp #include <vector>
#include <list> 
#include <stack> 
#include <iostream> 
#include <typeinfo> 

template <typename T> 
class ElementT; // primary template 

template <typename T> 
class ElementT<std::vector<T> > { // partial specialization
 public: typedef T Type; 
};
```

局部特化可以达到目的，但是太复杂。大多数情况下，类型函数和可应用类型（即容器中的类型）一起实现的，例如借助容器类型定义的成员类型value_type

```cpp
template <typename C> 
class ElementT { 
    public: typedef typename C::value_type Type; 
};
```