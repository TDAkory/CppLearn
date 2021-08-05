---
description: Deducing Types
---

# understand\_template\_type\_deduction

```cpp
// normal template
template<typename T>
void f(ParamType param);

// usage
f(expr);
```

During compilation, compilers use expr to deduce two types: one for T and one for ParamType.These types are frequently different, because ParamType often contains adornments, e.g., const or reference qualifiers.

There are three cases:

* ParamType is a pointer or reference type, but not a universal reference. \(Universal references are described later. At this point, all you need to know is that they exist and that they’re not the same as lvalue references or rvalue references.\)
* ParamType is a universal reference.
* ParamType is neither a pointer nor a reference.

## Case 1: ParamType is a Reference or Pointer, but not a Universal Reference

1. If expr’s type is a reference, ignore the reference part.
2. Then pattern-match expr’s type against ParamType to determine T.

```cpp
template<typename T>
void f(T& param); // param is a reference

int x = 27;            // x is an int
const int cx = x;      // cx is a const int
const int& rx = x;     // rx is a reference to x as a const int

f(x);  // T is int, param's type is int&
f(cx); // T is const int, param's type is const int&
f(rx); // T is const int, param's type is const int&
```

```cpp
template<typename T> 
void f(T* param);    // param is now a pointer

int x = 27;
const int *px = &x;

f(&x);     // T is int, param's type is int* 
f(px);     // T is const int, param's type is const int*
```

## Case 2: ParamType is a Universal Reference

* If expr is an lvalue, both T and ParamType are deduced to be lvalue references. That’s doubly unusual. First, it’s the only situation in template type deduction where T is deduced to be a reference. Second, although ParamType is declared using the syntax for an rvalue reference, its deduced type is an lvalue reference.
* If expr is an rvalue, the “normal” \(i.e., Case 1\) rules apply.

```cpp
template<typename T> 
void f(T&& param);    // param is now a universal reference

int x = 27;              // as before
const int cx = x;        // as before
const int& rx = x;       // as before

f(x);     // x is lvalue, so T is int&, param's type is also int&
f(cx);    // cx is lvalue, so T is const int&, param's type is also const int&
f(rx);    // rx is lvalue, so T is const int&, param's type is also const int&
f(27);    // 27 is rvalue, so T is int, param's type is therefore int&&
```

## Case3: ParamType is Neither a Pointer nor a Reference

1. As before, if expr’s type is a reference, ignore the reference part.
2. If, after ignoring expr’s reference-ness, expr is const, ignore that, too. If it’s volatile, also ignore that. \(volatile objects are uncommon. They’re generally used only for implementing device drivers\)

```cpp
template<typename T> 
void f(T param);    // param is now pass by value

int x = 27;              // as before
const int cx = x;        // as before
const int& rx = x;       // as before

f(x);     // T's and param's types are both int
f(cx);    // T's and param's types are again both int
f(rx);    // T's and param's types are still both int
```

It's important to recognize that const and volatile is ignored only for by-value-parameters.

```cpp
template<typename T>
void f(T param);          // param is still passed by value

const char* const ptr =   // ptr is const pointer to const object
     "Fun with pointers";

f(ptr);                   // pass arg of type const char * const
```

Here, the const to the right of the asterisk declares ptr to be const: ptr can’t be made to point to a different location, nor can it be set to null. \(The const to the left of the asterisk says that what ptr points to—the character string—is const, hence can’t be modified.\) When ptr is passed to f, the bits making up the pointer are copied into param. As such, the pointer itself \(ptr\) will be passed by value. In accord with the type deduction rule for by-value parameters, the constness of ptr will be ignored, and the type deduced for param will be const char\*, i.e., a modifiable pointer to a const character string. The constness of what ptr points to is preserved during type deduction, but the constness of ptr itself is ignored when copying it to create the new pointer, param.

## Array Arguments

```cpp
const char name[] = "J. P. Briggs"; // name's type is const char[13]
const char * ptrToName = name;      // array decays to pointer
```

These types \(`const char *` and `const char[13]`\) are not the same, but because of the array-to-pointer decay rule, the code compiles.

But what if an array is passed to a template taking a by-value parameter?

```cpp
template<typename T>
void f(T param); // template with by-value parameter

f(name); // what types are deduced for T and param? [const char *]
```

Because array parameter declarations are treated as if they were pointer parameters, the type of an array that’s passed to a template function by value is deduced to be a pointer type.

Although functions can’t declare parameters that are truly arrays, they can declare parameters that are references to arrays!

```cpp
template<typename T>
void f(T& param); // template with by-reference parameter

f(name); // pass array to f, T is const char [13], ParamType is const char (&)[13]
```

Interestingly, the ability to declare references to arrays enables creation of a template that deduces the number of elements that an array contains:

```cpp
// return size of an array as a compile-time constant. (The
// array parameter has no name, because we care only about
// the number of elements it contains.)
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept 
{ 
    return N; 
}
```

declaring this function constexpr makes its result available during compilation. That makes it possible to declare, say, an array with the same number of elements as a second array whose size is computed from a braced initializer:

```cpp
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };      // keyVals has 7 elements
int mappedVals[arraySize(keyVals)]; // so does // mappedVals
```

Of course, as a modern C++ developer, you’d naturally prefer a std::array to a built-in array:

```cpp
std::array<int, arraySize(keyVals)> mappedVals; // mappedVals' size is 7
```

As for arraySize being declared noexcept, that’s to help compilers generate better code.

## Function Arguments

```cpp
void someFunc(int, double);

template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);

f1(someFunc);    // param deduced as ptr-to-func; type is void (*)(int, double)

f2(someFunc);    // param deduced as ref-to-func; type is void (&)(int, double)
```

## Things to Remember

* During template type deduction, arguments that are references are treated as non-references, i.e., their reference-ness is ignored.
* When deducing types for universal reference parameters, lvalue arguments get special treatment.
* When deducing types for by-value parameters, const and/or volatile argu‐ ments are treated as non-const and non-volatile.
* During template type deduction, arguments that are array or function names decay to pointers, unless they’re used to initialize references.

