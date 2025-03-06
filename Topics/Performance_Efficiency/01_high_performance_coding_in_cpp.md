# C++高性能编程

> [C++ 高性能编程（全）](https://www.cnblogs.com/apachecn/p/18172912)

**零开销原则**：你不使用的东西，你就不需要付费；你使用的东西，你无法手工编码得更好

一些 C++ 的旧但强大的特性，这些特性与健壮性有关，而不是性能，很容易被忽视：值语义、const正确性、所有权、确定性销毁和引用。

> 在内部，引用是一个不允许为空或重新指向的指针；因此，当将其传递给函数时不涉及复制。

**C++的缺点**：长时间的编译时间和导入库的复杂性（直到 C++20，C++一直依赖于一个过时的导入系统）；缺乏提供的库（C++提供的几乎只是最基本的算法、线程，以及从 C++17 开始的文件系统处理，图形、用户界面、网络、线程、资源处理等只能依赖外部库）

## 基本C++技术

## `auto`自动类型推到

**在函数签名中使用 `auto`**

| 显式类型的传统语法 | 使用 `auto` 的新语法 |
|--------------------|----------------------|
| `int val() const { return m_; }` | `auto val() const { return m_; }` |
| `const int& cref() const { return m_; }` | `auto& cref() const { return m_; }` |
| `int& mref() { return m_; }` | `auto& mref() { return m_; }` |

**使用 `decltype(auto)` 进行返回类型转发**

```cpp
int val_wrapper() { return val(); }    // 返回 int
int& mref_wrapper() { return mref(); } // 返回 int&

auto val_wrapper() { return val(); }   // 返回 int
auto mref_wrapper() { return mref(); } // 也返回 int

decltype(auto) val_wrapper() { return val(); }   // 返回 int
decltype(auto) mref_wrapper() { return mref(); } // 返回 int&
```

**使用 `auto` 声明变量**

```cpp
auto i = 0;
auto x = Foo{};
auto y = create_object();
auto z = std::mutex{};     // OK si
```

在 `C++17` 中引入了保证的拷贝省略，语句`auto x = Foo{}`与`Foo x{}`是相同的；也就是说，语言保证在这种情况下没有需要移动或复制的临时对象。

使用`auto`定义变量的一个很大的优势是，永远不会留下未初始化的变量，因为`auto x;`不会编译。

`const auto&`表示，具有绑定到任何东西的能力。原始对象永远不会通过这样的引用发生变异。是潜在昂贵的对象的默认选择。

`auto&`来表示可变引用。只有在打算更改引用的对象时才使用可变引用。

`auto&&`被称为转发引用（也称为通用引用）。它可以绑定到任何东西，这对某些情况很有用。转发引用将像`const`引用一样，延长临时对象的生命周期。但与`const`引用相反，`auto&&`允许我们改变它引用的对象，包括临时对象。

便于使用的实践建议

* 基本类型和小的非基本类型：使用 const auto。
* 潜在昂贵的大型类型：使用 const auto&。
* 需要可变引用：使用 auto&。
* 转发代码：使用 auto&&。