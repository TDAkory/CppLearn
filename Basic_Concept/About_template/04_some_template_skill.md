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
    public:
        void exit();
};

template <tyepname T>
class Derived : Base<T> {
    public:
        void foo() {
            exit();     // 调用外部的exit()或出现错误
        }
};
```

上述例子中，foo()内部决定要调用哪一个exit()时，并不会考虑基类Base中定义的exit().

* 对于那些在基类中声明，并且依赖模板参数的符号（函数 or 变量），应该在它们前面使用`this->`或者`Base<T>::`
