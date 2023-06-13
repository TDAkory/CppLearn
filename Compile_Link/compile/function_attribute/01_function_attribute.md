# [Function Attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html)

- [IBM](https://www.ibm.com/docs/en/xl-c-and-cpp-aix/16.1?topic=compatibility-function-attributes)

几种写法格式：

```c
return_type __attribute__((attribute name)) function_declarator

__attribute__ ((attribute_name)) return_type function_declarator

return_type function_declarator __attribute__ ((attribute_name))
```

```c
int __attribute__((attribute_name)) func(int i);   //Form 1
__attribute__((attribute_name)) int func(int);     //Form 2
int func() __attribute__((attribute_name));        //Form 3
```

## alias

The alias function attribute causes the function declaration to appear in the object file as an alias for another symbol. This language feature provides a technique for coping with duplicate or cumbersome names.

```c
void __func2() { /* function body*/ };
void func1() __attribute__ ((alias("__func2")));
```

```cpp
extern "C" __func2() { /* function body*/ };
void func1() __attribute__ ((alias("__func2")));
```

The compiler does not check for consistency between the declaration of func1 and definition of __func2. Such consistency remains the responsibility of the programmer.

## always_inline

The always_inline function attribute instructs the compiler to inline a function. This function can be inlined when all of the following conditions are satisfied:

* The function is an inline function that satisfies any of the following conditions:
  * The function is specified with the inline or __inline__ keyword.
  * The option -qinline+<function_name> is specified, where function_name is the name of the function to be inlined.
  * C++ The function is defined within a class declaration.
* The function is not specified with the noinline or __noinline__ attribute.
* The number of functions to be inlined does not exceed the limit of inline functions that can be supported by the compiler.

## weak variable attribute

The weak variable attribute causes the symbol resulting from the variable declaration to appear in the object file as a weak symbol, rather than a global one. The language feature provides the programmer writing library functions with a way to allow variable definitions in user code to override the library declaration without causing duplicate name errors.

## section

The section function attribute specifies the section in the object file in which the compiler should place its generated code. The language feature provides the ability to control the section in which a function should appear.

```c
__attribute__ ((__section__("section-name")))
```

The section_name specifies a named section as a string literal, maximum length of 16 characters, not counting spaces. Spaces in the string are ignored.

The section variable attribute can be applied to a declaration or definition of the following types of variables:

- initialized or static global or namespace variables
- static local variables
- C++ only uninitialized global or namespace variables
- C++ only static structure or class member variables

