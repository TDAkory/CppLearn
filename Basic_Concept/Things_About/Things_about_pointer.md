# Things about Pointer

## 1. [When should I use raw pointers over smart pointers?](https://stackoverflow.com/questions/6675651/when-should-i-use-raw-pointers-over-smart-pointers)

The rule would be this - if you know that an entity must take a certain kind of ownership of the object, always use smart pointers - the one that gives you the kind of ownership you need. If there is no notion of ownership, never use smart pointers.

## 2. [Pointer-to-member operators: `.*` `->*`](https://learn.microsoft.com/en-us/cpp/cpp/pointer-to-member-operators-dot-star-and-star?view=msvc-170)

The pointer-to-member operators .* and ->* return the value of a specific class member for the object specified on the left side of the expression. The right side must specify a member of the class.

```cpp
// expre_Expressions_with_Pointer_Member_Operators.cpp
// compile with: /EHsc
#include <iostream>

using namespace std;

class Testpm {
public:
   void m_func1() { cout << "m_func1\n"; }
   int m_num;
};

// Define derived types pmfn and pmd.
// These types are pointers to members m_func1() and
// m_num, respectively.
void (Testpm::*pmfn)() = &Testpm::m_func1;
int Testpm::*pmd = &Testpm::m_num;

int main() {
   Testpm ATestpm;
   Testpm *pTestpm = new Testpm;

// Access the member function
   (ATestpm.*pmfn)();
   (pTestpm->*pmfn)();   // Parentheses required since * binds
                        // less tightly than the function call.

// Access the member data
   ATestpm.*pmd = 1;
   pTestpm->*pmd = 2;

   cout  << ATestpm.*pmd << endl
         << pTestpm->*pmd << endl;
   delete pTestpm;
}
```

In the preceding example, a pointer to a member, pmfn, is used to invoke the member function m_func1. Another pointer to a member, pmd, is used to access the m_num member.

The binary operator .* combines its first operand, which must be an object of class type, with its second operand, which must be a pointer-to-member type.

The binary operator ->* combines its first operand, which must be a pointer to an object of class type, with its second operand, which must be a pointer-to-member type.

In an expression containing the .* operator, the first operand must be of the class type of, and be accessible to, the pointer to member specified in the second operand or of an accessible type unambiguously derived from and accessible to that class.

In an expression containing the ->* operator, the first operand must be of the type "pointer to the class type" of the type specified in the second operand, or it must be of a type unambiguously derived from that class.

## 3. [C++异常-智能指针](https://mysteriouspreserve.com/blog/2022/04/08/Cpp-Exception-Smart-Pointer/)

智能指针可以保证最基本的异常安全，但是错误的使用仍会出问题。

下面的函数调用表达式，由于C++没有规定在函数调用表达式中，以何种顺序计算调用过程中需要用到的值，所以`new X`和`new Y`可能同时发生构造，`unique_ptr`无法起到保护作用。

```cpp
// Bad
foo(std::unique_ptr<X>(new X), std::unique_ptr<Y>(new Y));
// Good
foo(std::make_unique<X>(), std::make_unique<Y>());
```

## Name Counting and unique_ptr

> [Tip of the Week #55: Name Counting and unique_ptr](https://abseil.io/tips/55)

本文通过“**名字计数（Name Counting）**”这一简单直观的方法，解释`std::unique_ptr`的核心特性——**独占所有权**，并指导开发者正确使用`unique_ptr`以避免编译错误和运行时内存问题（如双删），同时阐明`std::move()`在“消除多余名字”、维护独占性中的关键作用。

文档中“名字”的严格界定是理解`unique_ptr`的前提：

- **定义**：值的“名字”指**任意作用域内的“值类型变量”**（非指针、非引用），该变量持有特定数据值；从C++标准角度，本质是“左值（lvalue）”。
- **示例**：`std::unique_ptr<Foo> g = NewFoo();`中，`g`就是`unique_ptr`所持指针值的“名字”；而`Foo* p = g.get();`中的`p`是指针变量，不属于“名字”范畴。

`std::unique_ptr`的核心设计目标是**独占内存所有权**，其行为严格遵循一条规则： 

**同一非空指针值，在任意执行时刻，只能对应一个“名字”（即只能存在于一个`std::unique_ptr`值类型变量中）**。  

违反此规则会导致两种问题：

1. **编译错误**：编译器直接拦截明显的“多名字”场景；
2. **运行时错误**：若绕过编译器检查（如直接用裸指针初始化多个`unique_ptr`），会触发“双删（double-delete）”，导致程序崩溃或内存错乱。

```cpp
std::unique_ptr<Foo> NewFoo() {
  return std::unique_ptr<Foo>(new Foo(1));
}

void AcceptFoo(std::unique_ptr<Foo> f) { f->PrintDebugString(); }

// the unique pointer allocated with NewFoo() only ever has one name by which you could refer it: the name “f” inside AcceptFoo()
void Simple() {
  AcceptFoo(NewFoo());
}

// the unique pointer allocated with NewFoo() has two names which refer to it: DoesNotBuild()’s “g” and AcceptFoo()’s “f”
void DoesNotBuild() {
  std::unique_ptr<Foo> g = NewFoo();
  AcceptFoo(g); // DOES NOT COMPILE!
}

void SmarterThanTheCompilerButNot() {
  Foo* j = new Foo(2);
  // Compiles, BUT VIOLATES THE RULE and will double-delete at runtime.
  std::unique_ptr<Foo> k(j);
  std::unique_ptr<Foo> l(j);
}
```

### 四、解决方案：用`std::move()`消除“多余名字”
当需要转移`unique_ptr`的所有权（如传递给函数、赋值给其他变量）时，需用`std::move()`实现“**名字擦除**”，确保始终只有一个有效名字：
1. **`std::move()`的作用**：概念上“擦除”原变量的“名字”属性——转移所有权后，原`unique_ptr`变量（如`h`）不再被视为该指针值的“名字”，后续不可再访问（直到重新赋值）。
2. **修复示例**（针对`DoesNotBuild()`）：
   ```cpp
   void EraseTheName() {
     std::unique_ptr<Foo> h = NewFoo();  // h是第一个名字
     AcceptFoo(std::move(h));            // 用std::move()擦除h的名字，仅AcceptFoo内的f是有效名字
   }
   ```
3. **关键约束**：调用`std::move()`后，必须避免访问原变量（如`h`），直到为其重新分配新的`unique_ptr`值，否则会访问“无效所有权”的变量，导致未定义行为。


### 五、“名字计数”的通用价值
“名字计数”不仅限于`std::unique_ptr`，还是现代C++的通用实用技巧：
- 辅助理解**移动语义**：判断是否存在“多余名字”（多余名字通常意味着不必要的拷贝，移动语义可消除这些拷贝）；
- 避免资源冲突：对所有“移动-only类型”（如`std::unique_ptr`、`std::future`），均可通过“计数名字”确保唯一所有权，避免资源竞争或重复释放；
- 简化复杂概念：无需深入理解“左值/右值”等底层语义，仅通过“数名字”即可快速判断代码是否合规。


### 六、核心总结（与原文完全一致）
1. **`unique_ptr`的核心规则**：同一非空指针值只能对应1个“名字”（值类型变量），多名字会导致编译/运行时错误；
2. **名字计数方法**：逐行统计持有同一指针值的`unique_ptr`“名字”数量，超过1个即违规；
3. **解决多名字问题**：用`std::move()`擦除原“名字”，转移所有权，确保始终唯一有效名字；
4. **关键提醒**：`std::move()`不“移动”数据，仅“擦除名字”并转移所有权；绕过编译器检查的多名字场景（如裸指针初始化多个`unique_ptr`），会触发运行时双删。