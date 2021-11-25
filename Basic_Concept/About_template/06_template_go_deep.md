# 深入模板基础

## 参数化声明

```cpp
template <typename T>
class List {            // 外部作用域的类模板
    public:
        template <typename T2> List(List<T2> const &);  // 成员函数模板
};

template <typename T>
    template <typename T2>
List<T>::List(List<T2> const &b) {}     // 位于类外部的成员函数模板定义

template <typename T>
int Length(List<T> const&);         // 外部作用域的函数模板

class Collection {
    template <typename T>
    class Node {            // 类内部的类模板定义

    };

    template <typename T>
    class Handle;           // 类内部的类模板声明

    template <typename T> T* alloc() {}     // 类内部的成员函数模板定义，显式内联
};

template <typename T>
class Collection::Handle {      // 在类外部定义的成员类模板

};
```

在所属外围类的外部定义的成员模板可以具有多个模板参数，由外向内依次指明模板的实参。

联合模板也是允许的，被看为类模板的一种

```cpp
template <typename T>
union AllocChunk {
    T object;
    unsigned char bytes(sizeof(T));
};
```