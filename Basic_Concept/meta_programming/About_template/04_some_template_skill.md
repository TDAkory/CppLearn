# 技巧性基础知识

## `typename`

```cpp
template <typename T>
class MyClass {
    typename T::SubType *ptr;
    ...
};
```

上面例子中，第二个`typename`说明，`SubType`是定义与类T内部的一个类型。如果不使用`typename`，`SubType`则会被认为是一个静态成员。

```cpp
    T::SubType *ptr;    // 会被看作是T的静态成员SubType和ptr的乘积
```

## `.template`构造

```cpp
template <int N>
void PrintBitSet(std::bitset<N> const &bs) {
    std::cout << bs.template to_string<char, char_traits<char>, allocator<char>>();
}
```

如果没有`template`，编译器不知道下面的事实：`bs.template`后面的小于号并不是数学含义的小于号，而是模板参数列表的其实符号。只有在编译器判断小于号之前，存在依赖于模板参数的构造，才会出现这种问题。上面例子中，传入参数bs就是依赖于模板参数`N`的构造。

## 使用`this->`

对于具有基类的类模板，自身使用名称x并不一定等同于this->x。即使x是从基类继承获得的，也是如此。例如：

```cpp
template <typename T>
class Base {
    
        void exit();
};

template <tyepname T>
class Derived : Base<T> {
    
        void foo() {
            exit();     // 调用外部的exit()或出现错误
        }
};
```

上述例子中，foo()内部决定要调用哪一个exit()时，并不会考虑基类Base中定义的exit().

* 对于那些在基类中声明，并且依赖模板参数的符号（函数 or 变量），应该在它们前面使用`this->`或者`Base<T>::`

## 成员模板

嵌套类和成员函数都可以作为模板。

通过定义一个作为模板的赋值运算符：针对元素类型可以转换的两个栈，就可以进行相互赋值

```cpp
template <typename T>       // 同样的，也可以将容器类型作为模板参数，这样就可以改变内部容器类型
class Stack {
    private:
        std::deque<T> elems;
    
    
        void push(T const&);
        ...
        // 使用元素类型为T2的栈赋值
        template <typename T2>
        Stack<T>& operator=(Stack<T2> cosnt&);
};

template <typename T>
template <typename T2>
Stack<T>& Stack<T>::operator=(Stack<T2> const &op2) {
    if ((void *)this == (void *)&op2) {
        return *this;
    }

    Stack<T2> tmp(op2);

    elems.clear();
    while(!tmp.empty()) {
        elems.push_font(tmp.top()); // 该句会发生类型检查，无法进行自动类型转换的类型间赋值仍然会报错
        tmp.pop();
    }
    return *this;
}
```

## 模板的模板参数

在`Stack`的例子中，如要使用一个与缺省值不同的内部容器，程序员必须两次指定元素类型。即，为了指定内部容器的类型，需要同时传递容器的类型和它所含元素的类型：

```cpp
Stack<int, std::vector<int>> vStack;
```

然而，借助于`模板的模板参数`，可以只指定容器的类型：

```cpp
Stack<int, std::vector> vStack;
```

为了获得该特性，第2个模板参数需要指定为`模板的模板参数`：

```cpp
template <typename T,
        template <typename ELEM> class CONT = std::deque >
class Stack {
    private:
        CONT<T> elems;
    
    
        void push(T const&);
        void pop();
        T top() const;
        bool empty() const {
            return elems.empty();
        }
};
```

不同之处在于，第2个参数被声明为类模板

```cpp
template <typename ELEM> class CONT
```

缺省值变为`std::deque`。在使用时，第2个参数必须是一个类模板，并且有第一个模板参数传递进来的类型进行实例化：`CONT<T> elems;`。

实际上，可以使用类模板内部的任何类型，来实例化模板的模板参数。

另外，这里没有使用到“模板的模板参数的”模板参数（即ELEM），因此可以省略，如下：

```cpp
template <typename T,
        template <typename> class CONT = std::deque >
class Stack {
    ...
};
```

对应的，成员函数的声明进行修改，如下：

```cpp
template <typename T, template <typename> class CONT>
void Stack<T, CONT>::push(T const &elem) {
    elems.push_back(elem);
}
```

最后，函数模板并不支持模板的模板参数。

## 模板的模板实参匹配

模板的模板参数（template template parameter）是指C++语言程序设计中，一个模板的参数是模板类型。只有类模板允许其模板参数是模板类型；函数模板不允许具有这样的模板参数。

模板的模板参数可以有默认值。例如：

```cpp
template <class T = float> struct B {};
template <template <class TT = float> class T> struct A {
     inline void f();
     inline void g();
};
template <template <class TT> class T> void A<T>::f() {
      T<> t; // error - TT has no default template argument
}
template <template <class TT = char> class T> void A<T>::g() {
       T<> t; // OK - T<char>
}
```

模板的模板参数，其实参应当是类模板名字或者别名模板（alias template）。当模板的模板参数做“形参实参结合”时，仅考虑把实参的基本（即未特化）类模板与模板的模板形参做匹配；不考虑实参的偏特化类模板，即使偏特化后的参数列表与模板的模板形参匹配得上。例如：

```cpp
template <typename T, template <typename ELEM, typename ALLOC = std::allocator<ELEM> > 
                         class CONT = std::deque>   //deque的基本型有2个模板参数 
class Stack { 
  private: 
    CONT<T> elems;         // OK！
    //… 
}; 
当模板的模板参数被实例化时，考虑采用该模板参数的偏特化版本。如果在实例化之处该偏特化版本仍不可见，且如果它是可见的则应该当采用，这时该程序为病态。[4] 例如:

template<class T> class A { // primary template
      int x;
};
template<class T> class A<T*> { // partial specialization
       long x;
};
template<template<class U> class V> class C {
        V<int> y;
        V<int*> z;
};
C<A> c; // V<int> within C<A> uses the primary template, so c.y.x has type int
        // V<int*> within C<A> uses the partial specialization, so c.z.x has type long
```

当一个模板的模板形参（称作P）与一个作为实参的模板（称作A）匹配时， 要求P与A各自的模板形参列表的对应成员满足：

* 具有相同种类（类型参数、非类型参数、模板作为参数）；
* 如果二者是非类型的模板参数，则其具体类型应相等;
* 如果二者是模板作为参数，则递归执行上述模板的模板参数形实匹配规则；
* 如果P是一个可变参数模板，即P的模板形参列表包含了模板参数包（template parameter pack），则这个模板参数包应该匹配A中0个或者多个模板参数或者匹配A中的对应的模板参数包。例如：

```cpp
    template<class T> class A { /* ... */ };
    template<class T, class U = T> class B { /* ... */ };
    template <class ... Types> class C { /* ... */ };
    template<template<class> class P> class X { /* ... */ };
    template<template<class ...> class Q> class Y { /* ... */ };
    X<A> xa; // OK
    X<B> xb; // ill-formed: default arguments for the parameters of a template argument are ignored
    X<C> xc; // ill-formed: a template parameter pack does not match a template parameter
    Y<A> ya; // OK
    Y<B> yb; // OK
    Y<C> yc; // OK  
    template <class T> struct eval;
    template <template <class, class...> class TT, class T1, class... Rest>
    struct eval<TT<T1, Rest...>> { };
    template <class T1> struct A;
    template <class T1, class T2> struct B;
    template <int N> struct C;
    template <class T1, int N> struct D;
    template <class T1, class T2, int N = 17> struct E;
    eval<A<int>> eA; // OK: matches partial specialization of eval
```

## 零初始化

对于int、double或指针等基本类型，不存在缺省构造函数，未初始化会带来未定义的值（undefined）。

在编写模板时，如果希望模板类型的变量都已经用缺省值初始化完毕，则会遇到问题，内建类型无法满足该要求：

```cpp
template <typename T>
void foo() {
    T x;    // 如果T是内建类型，x是一个不确定值
    
    T x = T();  // 如果T是内建类型，x是0或者false
}
```

类模板也需要定义一个缺省构造函数，通过初始化列表，来初始化类模板的成员：

```cpp
template <typename T>
class MyClass {
    private:
        T x;
    
        MyClass() : x() {}
        ...
};
```

## 使用字符串作为函数模板的实参

```cpp
template <typename T>
inline T const& max(T const &a, T const &b) {   // 注意：引用参数
    return a < b ? b : a;
}

template <typename T>
inline T min(T a, T b) {
    return a < b ? a : b;
}

int main() {
    std::string s;

    ::max("apple", "peach");    // ok: 相同类型
    ::max("apple", "tomato");   // error: 不同类型
    ::max("apple", s);          // error: 不同类型

    ::min("apple", "peach");    // ok: 相同类型
    ::min("apple", "tomato");   // ok: 退化相同类型
    ::min("apple", s);          // error: 不同类型
}
```

由于长度的区别，这些字符串属于不通过的数组类型，`apple`和`peach`是`char const[6]`, `tomato`是`char const[7]`;

但是在非引用类型的参数调用时，会发生数组到指针的自动类型转换（decay)。

## 小结

* 如果要访问依赖于模板参数的类型名称，应该在类型名称前添加`typename`
* 嵌套类和成员函数也可以是模板
* 赋值运算符的模板版本并没有取代缺省赋值运算符
* 类模板也可以作为模板参数，称之为模板的模板参数
* 模板的模板实参必须精确地匹配，匹配的时候不会考虑“模板的模板实参”的缺省模板参数（如std::deque的allocator）
* 通过显示调用缺省构造函数，确保模板的变量和成员得到一个缺省的初始值
* 对于字符串，在实参演绎的过程中，当且仅当参数不是引用时，才会出现数组到指针的类型转换
