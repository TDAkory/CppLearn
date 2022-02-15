# 实例化

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
