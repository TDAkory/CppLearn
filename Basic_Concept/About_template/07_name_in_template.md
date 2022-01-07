# 模板中的名称

- 1. 如果一个名称使用域解析运算符（即`::`）或者成员访问运算符（即 `.`或`－>`） 来显式表明它所属的作用域， 我们就称该名称为受限名称。 例如: `this->count`就是一个受限名称，而`count`则不是（即使前面没有符号，`count`实际上引用的也是一个类成员） 。

- 2. 如果一个名称（以某种方式） 依赖于模板参数，我们就称它为依赖型名称。例如，如果`T`是一个模板参数，`std::vector<T>::iterator`就是一个依赖名称；但如果`T`是一个已知的`typedef`（类型定义，例如int），那么`std::vector<T>::iterator`就不是一个依赖名称。

## 名称查找

```cpp
// 受限名称的名称查找是在一个受限作用域内部进行的， 该受限作用域由一个限定的构造所决定。 如果该作用域是一个类， 那么查找范围可以到达它的基类； 但不会考虑它的外围作用域。

int x;
class B {
public:
    int i;
}; 
 
class D : public B {
};

void f(D* pd) {
    pd->i = 3; // 找到B::i
    D::x = 2;  // 错误： 并不能找到外围作用域中的::x
}
```

```cpp
// 非受限名称的查找则相反， 它可以（由内到外） 在所有外围类中逐层地进行查找（但在某个类内部定义的成员函数定义中， 它会先查找该类和基类的作用域， 然后才查找外围类的作用域） ， 这种查找方式也被称为普通查找。
extern int count;               // (1)
int lookup_example(int count)   // (2)
{
    if (count < 0) 
    {
        int count = 1;          // (3)
        lookup_example(count);  // 非受限的count将会引用（3）
    } 
    return count + ::count;     // 第1个(非受限的)count引用(2),第2个（受限的）count引用（1）
}
```

### ADL(Argument-Dependent Lookup)

对于非受限名称的查找，最近增加了一项新的查找机制——除了前面的普通查找——就是说非受限名称有时可以使用依赖于参数的查找。

ADL只能应用于非受限名称。在函数调用中，这些名称看起来像是非成员函数。对于成员函数名称或者类型名称，如果普通查找能找到该名称，那么将不会应用ADL。如果把被调用函数的名称用圆括号括起来，也不会使用ADL。否则，如果名称后面的括号里面有（一个或多个）实参表达式，那么**ADL将会查找这些实参的associated class（关联类）和associated namespace（关联名字空间）**，稍后给出二者的定义。但从直观上来看， 我们可以认为是：与给定类型直接相关的所有namespace和class。 

例如，如果某一类型是指向class X的指针，那么它的associated class和associated namespace会包含X和X所属的任何class和namespace。

**对于给定类型，对于由associated class和associated namespace所组成的集合的准确定义，我们可以通过下列规则来确定：**

- 对于基本类型，该集合为空集。
- 对于指针和数组类型，该集合是所引用类型（譬如对于指针而言，它所引用的类型是“指针所指对象”的类型）的associated class和associated namespace。
- 对于枚举类型，associated namespace指的是枚举声明所在的namespace。
- 对于类成员，associated class指的是它所在的类。
- 对于class类型（包含联合类型），associated class集合包括：该class类型本身、它的外围类型、直接基类和间接基类。 associated namespace集合是每个associated class所在的namespace。 如果这个类是一个类模板实例化体， 那么还包含：模板类型实参本身的类型、声明模板的模板实参所在的class和namespace。
- 对于函数类型，该集合包括所有参数类型和返回类型的associated class 和associated namespace。
- 对于类X的成员指针类型，除了包括成员相关的associated anmespace和associated calss，该集合还包括与X相关的associated namespace和associated class。

**ADL会在所有的associated class和associated namespace中依次地查找，就好像依次地直接使用这些名字空间进行限定一样。唯一的例外情况是：它会忽略using指示符**


```cpp
namespace X {
    template<typename T> void f(T);
} 
namespace N{
    using namespace X;
    enum E { e1 };
    void f(E) {
        std::cout << "N::f(N::E) called\n";
    }
} 
 
void f(int)
{
    std::cout << "::f(int) called\n";
} 
 
int main()
{
    ::f(N::e1);     // 受限函数名称：不会使用ADL
    f(N::e1);       // 普通查找将找到f(); ADL将找到N::f(),将会调用后者
}
```

### 友元名称插入

在类中的友元函数声明可以是该友元函数的首次声明。在此前提下，对于包含这个友元函数的类，假设它所属的最近名字空间作用域（可能是全局作用域）为作用域A，我们就可以认为该友元函数是在作用域A中声明的。这里， 我们会遇到一个颇有争议的话题：在插入友元声明的（类）作用域中，该友元声明是否应该是可见的呢？ 实际上，多数情况下只有在模板中才会出现这个问题，考虑下面的例子：

```cpp
template <typename T>
class C 
{
    ...
    friend void f();
    friend void f(C<T> const&); // 插入式类名称，定义见8.3 模板实参
    ...
};
void g(C<int>* p)
{
    f();    // f()在此是可见的吗   
    f(*p);  // f(C<int> const&)在此是可见的吗
}
```

这里的问题是：如果友元声明在外围类中是可见的， 那么实例化一个类模板可能会使一些普通函数（如 f() ）的声明也成为可见的。 一些程序员会认为这样很出乎意料。 因此 C++标准规定：**通常而言，友元声明在外围（类） 作用域中是不可见的(也就是说，出了声明所在的作用域，就不可见了。对于上例子，在class C以外，f是不可见的)。**然而，存在一个有趣的编程技术，它依赖于只在友元声明中声明（或者定义）某个函数（即Barton-Nackman方法，后续章节说明）。因此C++标准还规定：**如果友元函数所在的类属于ADL的关联类集合， 那么我们在这个外围类是可以找到该友元声明的。**这就意味着：函数调用实参必须和包含友元函数的类具有关联；**如果调用实参和包含友元函数的类不具备关联关系，那么将找不到该友元函数。**请看下面的例子：

```cpp
class S {
};
 
template<typename T>
class Wrapper 
{
private:
    T object;
public:
    Wrapper(T obj) : object(obj) { // 可以把T隐式转型为Wrapper<T>
    }
    friend void f(Wrapper<T> const& a) {
    }
};
int main()
{
    S s;
    Wrapper<S> w(s); 
    f(w);   // 正确：Wrapper<S>是一个和w相关联的类。
    f(s);   // 错误：Wrapper<S>和s不相关联。
}
```

在这个例子中，调用`f(w)`是有效的，因为函数`f()`是一个在`Wrapper<S>`内部进行声明的友元函数，而 `Wrapper<S>`和实参`w`是相关的。 然而，在调用 f(s)中，友元函数的声明`f(Wrapper<S> const&)`就不是可见的， 因为类`Wrapper<S>`和实参`s`的类型`S`是不相关联的。 因此，尽管存在一个从`S`到`Wapper<S>`的有效隐式转型（借助于`Wrapper<S>`的构造函数），但此时并不会考虑这种转型，因为编译器并不能首先找到候选函数`f`， 当然也就不会考虑`f`的参数所要进行的转型了。

再次考虑上面的包含函数`f()`的例子。 调用`f()`并没有关联类或者名字空间， 因为它没有任何参数，不能利用ADL，因此是一个无效调用。 然而，`f(*p)`具有关联类`C<int>`（因为*p的类型是`C<int>`）；因此， 只要我们在调用之前完全实例化了类`C<int>`， 就可以找到第2个友元函数（即f）声明。为了确保这一点，我们可以假设： **对于涉及在关联类中友元查找的调用， 实际上会导致该（关联）类被实例化（如果还没有实例化的话）。**


### 插入式类名称

如果在类本身的作用域中插入该类的名称，则称其为插入式类名称。它可以被看作位于该类作用域中的一个非受限名称，而且是可访问的名称。

```cpp
int C;

class C {
private:
    int i[2];

public:
    static int f() {
        return sizeof(C);
    }
};

int f() {
    return sizeof(C);
}

int main() {    // 成员函数C::f()返回类型C的大小，而函数::f()返回变量C的大小
    std::cout << "C::f() = " << C::f() << ", "
              << "::f() = " << ::f() << std::endl;
}
```

类模板也可以具有插入式类名称。

```cpp
template <template<typename> class TT> class X {};

template <typename T> class C {
    C *a;       // right, 等价于 C<T> *a
    C<void> b;  // right
    X<C> c;     // wrong, 后面没有模板实参列表的C不被看作模板
    X<::C> d;   // wrong <:是[的另一种标记
    X< ::C> e;  // 正确 在< 和 ::之间的空格是必须的
};
```

### 解析模板