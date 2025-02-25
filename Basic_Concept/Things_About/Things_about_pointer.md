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