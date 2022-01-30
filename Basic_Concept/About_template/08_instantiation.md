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