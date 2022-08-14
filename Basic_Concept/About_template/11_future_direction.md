# Future Direction

## 编译器对连续尖括号的支持

低版本编译器会混淆模板的尖括号和右移运算符，需要在使用时添加空格，高版本编译器则没有这项困扰

```cpp
typedef std::vector<std::list<int> > LineTable;     // 低版本编译器正确

typedef std::vector<std::list<int>> OtherTable;     // 低版本错误、高版本正确
```

## 更宽松的typename使用

```cpp
template <typename T> 
class Array { 
    public: 
    typedef T ElementT; 
    …
};

template <typename T> 
void clear (typename Array<T>::ElementT& p); // OK

template<> 
void clear (typename Array<int>::ElementT& p); // ERROR
```

语言的设计者认为：对于任何前面没有使用关键字struct、class、union或者enum之一的受限类型名称，都可以在前面加上关键字typename。

## 缺省函数模板参数

一个约束：如果模板参数具有一个缺省实参值，那么位于该参数后面的每个参数都必须具有缺省模板实参。这个约束也同样适用于类模板。

## 字符串和浮点型的模板实参

在非类型模板实参的所有约束中，令模板初学者和高级用户感到最意外的可能就是：不能让字符串文字作为模板实参。

```cpp
template <char const* msg> 
class Diagnoser { 
    public: 
    void print(); 
}; 

int main() 
{ 
    Diagnoser<"Surprise!">().print(); 
}
```

Diagnoser<“X”>和Diagnoser<“X”>实际上是两个不同的实例，同时也就属于不同的类型（我们看到：“X”的实际类型是char const[2]，但是当把它传递给模板实参的时候，它会decay成char const*类型）。

浮点类型作为非类型模板参数是可行的。

```cpp
template<double Ratio>
class Converter {
    static double convert(double val) {
        return val * Ratio;
    }
};

typedef Converter<0.0254> InchToMeter;
```

## 放松模板模板参数的匹配

用于替换模板的模板参数的模板必须能够和模板参数的参数列表精确地匹配，但这有时候会导致意外的结果。

```cpp
#include <list> 
 // declares:
 // namespace std { 
 // template <typename T, 
 // typename Allocator = allocator<T> > 
 // class list; 
 // } 
template<typename T1, 
         typename T2, 
         template<typename> class Container> 
 // Container expects templates with only one parameter
class Relation { 
 public: 
 …
 private: 
    Container<T1> dom1; 
    Container<T2> dom2; 
}; 
int main() 
{ 
    Relation<int, double, std::list> rel; 
    // ERROR: std::list has more than one template parameter
 …
}
```

## typedef模板

```cpp
template<typename T>
typedef vector<List<T>> Tables;

Tables<int> t;
```

## 函数模板的局部特化

* 在不改变类定义的前提下，我们可以特化勒种的某个成员模板。然而，如果要给类增加一个重载函数，我们就不得不改变这个类的定义
* 为了重载函数模板，多个重载函数之间的参数必须由本质上的区别
* 哪些针对某个非重载函数的合法代码，在对这个函数进行重载之后，就可能会变成不合法的代码。
* 针对引用“特定函数模板或者特定函数模板的实例化实体”的友元声明，函数模板的重载版本并不能自动获得原始版本的特权

```cpp
template<typename T>
T const& max(T const&, T const&);   // primary template

template<typename T>
T* const& max(T* const&, T* const&);    // partial specialization
```

## typeof运算符

## 命名模板实参

## 静态属性

## 客户端实例化信息

## 重载类模板

## List参数

## 布局控制

## 初始化器的演绎

## 函数表达式
