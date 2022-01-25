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