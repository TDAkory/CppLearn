# Exception-Safe code in C++

> [Exception-Safe Coding in C++ By Jon Kalb](http://www.exceptionsafecode.com/)
>
> [Part 1](https://www.youtube.com/watch?v=N9bR0ztmmEQ&t=45s) |
> [Part 2](https://www.youtube.com/watch?v=UiZfODgB-Oc)

- [Exception-Safe code in C++](#exception-safe-code-in-c)
  - [What's the Problem?](#whats-the-problem)
  - [Solutions without Exception](#solutions-without-exception)
  - [First Steps](#first-steps)
    - [Wrong way](#wrong-way)
    - [Right way](#right-way)
    - [Exception-Safety Guarantees(Abrahams)](#exception-safety-guaranteesabrahams)
  - [Mechanics](#mechanics)
    - [**Guidelines From Below**](#guidelines-from-below)
    - [How exceptions work in C++](#how-exceptions-work-in-c)
      - [Error detection / throw](#error-detection--throw)
      - [Error handling / catch](#error-handling--catch)
    - [Performance Cost of try/catch](#performance-cost-of-trycatch)
    - [Function Try Blocks](#function-try-blocks)
    - [Function Try Block for a Constructor](#function-try-block-for-a-constructor)
    - [New in C++11](#new-in-c11)
    - [noexcept](#noexcept)
    - ["Terminate" Handler](#terminate-handler)
      - [How to not "Terminate"](#how-to-not-terminate)
    - [Safe Objects](#safe-objects)
      - [Object Lifetimes](#object-lifetimes)
      - [Aborted Construction](#aborted-construction)
    - [RAII(Resource Acquisition Is Initialization)](#raiiresource-acquisition-is-initialization)
      - [What happens to the object if acquisition fails?](#what-happens-to-the-object-if-acquisition-fails)
      - [RAII Cleanup](#raii-cleanup)
    - [swap()](#swap)

## What's the Problem?

- Seperation of Error Detection from Error Handling

## Solutions without Exception

1.1 Error Flagging

- errno
- "GetRrror" Function

1.2 Problems with the Error Flagging Approach

- Errors can be ignored
  - Errors are ignored by default
- Ambiguity about which call failed
- Code is tedious to read and write

2.1 Return Code

- Almost every APU returns a code
- Usually int or long
- Known set of error/status values
- Error codes relayed up the call chain

2.2 Problems with the Return Code Approach

- Errors can be ignored
  - Are ignored by default
  - if a single call "break the chain" by not returning an error, errors cases are lost
- Code is tedious to read and write
- Exception based coding addresses both of these issues

## First Steps

### Wrong way

- Carefully check return values/error codes to detect and correct problems
- identify functions that can throw and think about what to do when they fail
- Use exception specifications so the compiler can help create safe code
- Use try/catch blocks to control code flow
**"You must unlearn what you have learned"  ---Yoda**

### Right way

- Think structurally
- Maintain invariants

### Exception-Safety Guarantees(Abrahams)

- Basic: invariants of the component are preserved, and no resources are leaked
- Strong: if an exception is thrown there are no effects
- No-Throw: operation will not emit an exception

## Mechanics

### **Guidelines From Below**

- Throw by value, Catch by reference
- Do not use dynamic exception specifications
- Destructors must not throw
  - Mst deliver the No-Throw Guarantee
  - Cleanup must always be safe
  - May throw internally, but may not emit
- Assign ownership of every resource immediately upon allocation, to a named manager object that manages no other resources
- Use RAII
  - Every responsibility is an object
  - One responsibility per object
- All cleanup code is called from a destructor
  - An object with such a destructor must be put on the stack as soon as calling the cleanup code become a responsibility
- Create `swap()` for value classes
  - Must deliver the No-Throw guarantee

### How exceptions work in C++

#### Error detection / throw

```cpp
{   // a runtime error is detected
    ObjectType object;
    throw object;
}
```

Is object thrown?           Must be copyable.
Can we throw a pointer?     No, unsafe
Can we throw a reference?   technically yes, but don't

```cpp
{
    std::string s("This is a local string.");
    throw ObjectType(constructor parameters);
}
```

#### Error handling / catch

```cpp
try {
    code_that_might_throw();
} catch (A a) {
    error_handling_code_that_can_use_a(a);
} catch (...) {
    more_generic_error_handling_code();
}
more_code();
```

```cpp
// what can catch A
try {
    throw A();
} 
catch (B) {}    // B is a public base class of A
catch (B &) {}
catch (B const &) {}
catch (B volatile &) {}
catch (B const volatile &) {}
catch (A) {}
catch (A &) {}
catch (A const &) {}
catch (A volatile &) {}
catch (A const volatile &) {}
catch (void *) {}   // if A is a pointer
catch (...) {}
```

### Performance Cost of try/catch

- No throw --- no cost
- In the throw case...
  - Don't know, Don't care.

### Function Try Blocks

```cpp
void F(int a) {
    try {
        int b;
        ...
    } catch (std::exception const &ex) {
        // can reference a, but not b
        // can throw, return, end
    }
}

// wrap the body of function in try
void F(int a) 
try {
    int b;
    ...
} catch (std::exception const &ex) {
    // can reference a, but not b
    // can throw, but can't return
}
```

### Function Try Block for a Constructor

Only use in to change the exception thrown by the constructor of a base class or data member constructor

```cpp
Foo::Foo(int a)
try:
Base(a);
member(a)
{
}
catch (std::exception &ex) {
    // Can reference a, but not Base or member
    // Can modify ex or throw a different exception
    // but an exception will be thrown, can't recover
}
```

### New in C++11

- Moving exceptions between threads
  - Capture thw exception
  
    ```cpp
    #include <exception>

    exception_ptr current_exception() noexcept;
    ```

    - `std::exception_ptr` is copyable
    - The `exception` exists as long as any `std::exception_ptr` using to it does
    - Can be copied between `thread` like any other data

    ```cpp
    std::exception_ptr ex(nullptr);
    try {
        ...
    } catch (...) {
        ex = std::current_exception();
        ...
    }
    if (ex) {
        ...
    }
    ```

  - Move the exception list any other object
  - Re-throw whenever we want

    ```cpp
    #include <exception>

    [[noreturn]] void rethrow_exception(exception_ptr p);
    ```

```cpp
int Func(); // might throw

std::future<int> f = std::async(Func());

int v(f.get()); // if Func() threw, it comes out here
```

- Nesting exceptions
  - Nesting the current exception

    ```cpp
    #include <exception>
    class nested_exception; //  Constructor implicitly calls current_exception() and holds the result
    ```

  - Throwing a new exception with the nested one

    ```cpp
    [[noreturn]] template<class T>
    void throw_with_nested(T &&t);  // Throws a type that is inherited from both T and std::nested_exception
    ```

  - Re-throwing just the nested one

```cpp
try {
    try {
        ...
    } catch(...) {
        std::throw_with_nested(MyException());
    }
} catch (MyException &ex) {
    // handle ex
    // Check if ex is a nested exception
    // Extract the contained exception
    // throw the contained exception
    std::rethrow_if_nested(ex);     // One call does all these steps
}
```

### noexcept

- noexcept specification (of a function) 

    > [noexcept异常说明及其使用](https://blog.csdn.net/qianqin_2014/article/details/51321631)

    ```cpp
    void F();   // may throw anything
    void G() noexcept(Boolean constexpr);    
    void G() noexcept;  // default to noexcept(true)
    ```

    Destructors are noexcept by default

- noexcept operator

    ```cpp
    static_assert(noexcept(2+3), "");   // true
    static_assert(not noexcept(throw 23), "");  //true
    inline int Foo() { return 0;}
    static_assert(noexcept(Foo()), ""); // false

    inline int Foo() noexcept { return 0;}
    static_assert(noexcept(Foo()), ""); // true
    ```

### ["Terminate" Handler](https://en.cppreference.com/w/cpp/error/terminate_handler)

> [std::set_terminate](https://en.cppreference.com/w/cpp/error/set_terminate)

- Called when re-throw and there is no exception
- Called when a "noexcept" function throws
- Called when throwing when there is already an exception being thrown
- Called on un-handling exception

#### How to not "Terminate"

- Don't re-throw outside of a catch block
- Don't throw from a "noexcpt" function
- Don't throw when an exception is being thrown

### Safe Objects

#### Object Lifetimes

- Order of construction:
  - Base class objects
    - As listed in the type definition, left to right
  - Data members
    - As listed in the type definition, top to bottom
    - Not as listed in the constructor’s initializer list
  - Constructor body
- Order of destruction:
  - Exact reverse order of construction

#### Aborted Construction

- How?
  - Throw from Constructor of base class, constructor of data member, constructor body
- What do we need to clean up?
  - Base class objects? NO
  - Data members? NO
  - Constructor Body? YES
    - We need to clean up anything we do here because the destructor will not be called, as long as object lifetime begins at the end of constructor.
  - What about new array?
    - like above, complete constructor

### [RAII(Resource Acquisition Is Initialization)](https://en.cppreference.com/w/cpp/language/raii)

#### What happens to the object if acquisition fails?

- The object never exists
- If you have the object, you have the resource
- If the attempt to get the resource failed, then the constructor threw and we don't have the object

#### RAII Cleanup

- Destructors have resource release responsibility
- Some objects may have a "release" member function
- Cleanup cannot throw

### swap()

- No Throw swapping is a key exception-safety tool

```cpp
struct Bight {
  ...
  void swap(Bight &) {  // No Throw
    // swap bases, then members
  } 
};

namespace std {
  template<> void swap<Bight>(Bight &a, Bight &b) {
    a.swap(b);
  }
}
```

- `std::swap<>()` is always an option
  - But it doesn't promise No-Throw
    - It does three copies --- Copies can fail
- Pur custom swaps can be No Throw
  - Don't use non-swapping base / member classes
  - Don't use const of reference data members
    - These are not swap-able

