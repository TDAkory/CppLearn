# [What are const rvalue references good for?](https://codesynthesis.com/~boris/blog//2012/07/24/const-rvalue-references/)

在日常代码中很少剪刀 `const 右值引用`，右值引用的主要目的是允许我们移动对象而不是复制它们，移动对象就意味着对原对象的修改，就是非 `const` 的。因此，移动构造函数和移动赋值运算符的入参都是非常量右值引用。

但 `const右值引用` 在语言标准中有明确定义的绑定规则。具体的，`const右值`更愿意绑定到 `const右值引用` 而不是 `const左值引用`。

```cpp
struct s {};
 
void f (      s&);  // #1
void f (const s&);  // #2
void f (      s&&); // #3
void f (const s&&); // #4
 
const s g ();
s x;
const s cx;
 
f (s ()); // rvalue        #3, #4, #2
f (g ()); // const rvalue  #4, #2
f (x);    // lvalue        #1, #2
f (cx);   // const lvalue  #2
```

这里存在不对称性：`const左值引用`可以绑定到`右值`，但 `const右值引用`不能绑定到`左值`。

即 `const右值引用`通常毫无用处。在不可修改引用上，`const左值引用`可以完美的胜任工作。对一个`右值引用`，`const`语义本身就是不应当存在的。

因为，`const右值引用`的唯一用途似乎是，如果您需要完全禁用右值引用，并且您需要以一种防弹的方式来处理上述毫无意义但仍然合法的情况。

Case in point is the `std::reference_wrapper` class template and its `ref()` and `cref()` helper functions. Because reference_wrapper is only meant to store references to lvalues, the standard disables `ref()` and `cref()` for rvalues. To make sure they cannot be called even for const rvalues, the standard uses the const rvalue reference:

```cpp
template <class T> void ref (const T&&) = delete;
template <class T> void cref (const T&&) = delete;
```

除了支持移动语义外，右值引用语法也用于完美转发。但请注意，添加consttoT&&会禁用用于支持此功能的特殊参数推导规则。

```cpp
template <typename T> void f (T&&);
template <typename T> void g (const T&&);
 
s x;
 
f (s ()); // Ok, T = s
f (x);    // Ok, T = s&
 
g (s ()); // Ok, T = s
g (x);    // Error, binding lvalue to rvalue, T = s
```