# propagate_const

> The matters you reflect upon have been previously deliberated by others.

常写C++的都知道，const用来声明变量或对象的不变性，但当涉及到指针或类指针对象的时候，这种不变性的约束似乎被打破了。

```cpp
// ptr 本身不能被修改
const std::unique_ptr<int> ptr = std::make_unique<int>(42);
// ptr 指向的对象（即 int 类型的值）仍然可以被修改，这是合法的
*ptr = 100;
```

更进一步的一个例子（[godbolt](https://godbolt.org/z/rs4v8f3W1)）

```cpp
struct A
{
  void bar() const 
  { 
    std::cout << "bar (const)" << std::endl; 
  }
  
  void bar() 
  { 
    std::cout << "bar (non-const)" << std::endl; 
  }
};

struct B
{
  B() : m_ptrA(std::make_unique<A>()) {} 
  
  void foo() const 
  { 
    std::cout << "foo (const)" << std::endl;
    m_ptrA->bar(); 
  }           
  
  void foo() 
  { 
    std::cout << "foo (non-const)" << std::endl;
    m_ptrA->bar(); 
  }

  std::unique_ptr<A> m_ptrA;
};

int main()
{    
  B b;
  b.foo();
  
  const B const_b;
  const_b.foo();
}
```

得到的输出是：

```
foo (non-const)
bar (non-const)
foo (const)
bar (non-const)
```

上述行为可以通过使用 `const_cast` 重写 `void B::foo() const` 来修正，以显式调用 `A` 的 `const` 成员函数。但这样的改动并不优雅。

有其他办法么？**`std::experimental::propagate_const`**

它来自于2015年的一份提案：[A Proposal to Add a Const-Propagating Wrapper to the Standard Library](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4372.html)

简言之：`std::experimental::propagate_const` 是一个包装器类，用于确保 `const` 属性不仅应用于指针或类指针对象本身，还应用于其所指向的对象。从而达到在 `const` 对象中传播 `const` 属性的功能。

还是用示例来描述：

**示例一**（[godbolt](https://godbolt.org/z/qa8n1jK1b)）

```cpp
#include <experimental/propagate_const>
#include <iostream>
#include <memory>

int main() {
    std::experimental::propagate_const<std::unique_ptr<int>> ptr = std::make_unique<int>(42);

    // 通过 const 访问路径访问时，ptr 被视为指向 const 的指针
    const auto& constPtr = ptr;
    std::cout << *constPtr << std::endl;  // 输出：42
    * constPtr = 50;        // 不合法，会有编译错误

    // 修改 ptr 指向的对象
    *ptr = 100;
    std::cout << *ptr << std::endl;  // 输出：100

    return 0;
}
```

会得到一个编译错误

```shell
# 编译输出
<source>: In function 'int main()':
<source>:11:16: error: assignment of read-only location '(& constPtr)->std::experimental::fundamentals_v2::propagate_const<std::unique_ptr<int> >::operator*()'
   11 |     * constPtr = 50;
      |     ~~~~~~~~~~~^~~~
Compiler returned: 1
```

**示例二**（[godbolt](https://godbolt.org/z/T3G46xajW)）

```cpp
#include <experimental/propagate_const>
#include <iostream>
#include <memory>
 
struct X
{
    void g() const { std::cout << "X::g (const)\n"; }
    void g() { std::cout << "X::g (non-const)\n"; }
};
 
struct Y
{
    Y() : m_propConstX(std::make_unique<X>()), m_autoPtrX(std::make_unique<X>()) {}
 
    void f() const
    {
        std::cout << "Y::f (const)\n";
        m_propConstX->g();
        m_autoPtrX->g();
    }
 
    void f()
    {
        std::cout << "Y::f (non-const)\n";
        m_propConstX->g();
        m_autoPtrX->g();
    }
 
    std::experimental::propagate_const<std::unique_ptr<X>> m_propConstX;
    std::unique_ptr<X> m_autoPtrX;
};
 
int main()
{
    Y y;
    y.f();
 
    const Y cy; // Y 的 const 属性，自然的传递到了其内部变量 m_propConstX 上
    cy.f();
}
```

得到输出

```
Y::f (non-const)
X::g (non-const)
X::g (non-const)
Y::f (const)
X::g (const)
X::g (non-const)
```

`std::experimental::propagate_const`的一些特性：

* 如果 `propagate_const` 包装的对象是 `const`，则其指向的对象也会被视为 `const`
* 如果 `propagate_const` 包装的对象是非 `const`，则其指向的对象也是非 `const`
* `propagate_const` 支持包装智能指针（如 `std::unique_ptr` 和 `std::shared_ptr`），也支持包装普通指针
* 对指针指向的访问是透明的：`propagate_const` 重载了 `operator*` 和 `operator->`，使得访问包装的对象与直接访问指针一样方便

`std::experimental::propagate_const`的注意事项：

* `propagate_const` 目前是 C++ 标准库的一个扩展，定义在 `<experimental/propagate_const>` 中），尚未正式纳入 C++ 标准
* 需要确认编译器的支持情况，通常在C++17或更高版本的编译器上可用
* `propagate_const` 是一个轻量级包装器，通常不会引入额外的性能开销

`std::experimental::propagate_const`的使用场景：

* 确保对象及其成员（特别是成员中包含指针指向的间接成员）都是不可变时
  
**参考引用：**

* [std::experimental::propagate_const](https://en.cppreference.com/w/cpp/experimental/propagate_const)