# understand-auto-type-deduction

When a variable is declared using auto, auto plays the role of T in the template, and the type specifier for the variable acts as ParamType. This is easier to show than to describe, so consider this example:

```cpp
auto x = 27;           // the type specifier for x is simply auto by itself.
const auto cx = x;     // the type specifier is const auto.
const auto& rx = x;    // the type specifier is const auto&.
```

To deduce types for x, cx, and rx in these exam‐ ples, compilers act as if there were a template for each declaration as well as a call to that template with the corresponding initializing expression.

Deducing types for auto is, with only one exception \(which we’ll discuss soon\), the same as deducing types for templates.

* Case 1: The type specifier is a pointer or reference, but not a universal reference.
* Case 2: The type specifier is a universal reference.
* Case 3: The type specifier is neither a pointer nor a reference.

```cpp
// case 1 & case 3
auto x = 27;
const auto cx = x;
const auto &rx = x;

// case 2
auto &&uref1 = x;     // x is lvalue, uref1 is int&
auto &&uref2 = cx;    // cx is const & lvalue, uref2 is const int &
auto &&uref3 = 27;    // 27 is rvalue, uref3 is int&&
```

Array and function names can decay into pointers for non-reference type specifiers, which happens in auto type deduction.

```cpp
const char name[] = "R. N. Briggs";    // name is const char[13]
auto arr1 = name;     // arr1 is const char *
auto &arr2 = name;    // arr2 is const char (&)[13]

void someFunc(int ,double)    // someFunc is void(int, double)
auto func1 = someFunc;        // func1 is void (*)(int, double)
auto &func2 = someFunc;       // func2 is void (&)(int, double)
```

Mostly, auto type deduction works like template type deduction. Except for the one way they differ.

```cpp
// declare an int with an initial value of 27
// c++98
int x1 = 27;
int x2(27);
// c++11
int x3 = {27};
int x4{27};

// all works fine


// for auto
auto x1 = 27;        // type is int, value is 27
auto x2(27);         // ditto
auto x3 = {27};      // type is std::initializer_list<int>, value is {27}
auto x4{27}          // ditto
```

When the initializer for an auto-declared variable is enclosed in braces, the deduced type is a std::initializer\_list. If such a type can’t be deduced \(e.g., because the values in the braced initializer are of different types\), the code will be rejected, because the types in brace are different:

```cpp
auto x5 = {1, 2, 3.0}; // error! can't deduce for std::initializer_list<int>
```

```cpp
auto x = { 11, 23, 9 };     // x's type is std::initializer_list<int>

template<typename T>
void f(T param);         // template with parameter declaration equivalent to x's declaration

f({ 11, 23, 9 });        // error! can't deduce type for T


// 对模板显式声明初始化列表，则可以推导成功
template<typename T>
void f(std::initializer_list<T> initList);

f({1, 2, 3});        // T is int, and initList is initializer_list<int>
```

**Things to Remember**

• auto type deduction is usually the same as template type deduction, but auto type deduction assumes that a braced initializer represents a std::initializer\_list, and template type deduction doesn’t.

• auto in a function return type or a lambda parameter implies template type deduction, not auto type deduction.

