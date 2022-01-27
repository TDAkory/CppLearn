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
    - [How exceptions work in C++](#how-exceptions-work-in-c)
      - [Error detection / throw](#error-detection--throw)
      - [Error handling / catch](#error-handling--catch)
    - [**Guideline-1**](#guideline-1)
    - [Performance Cost of try/catch](#performance-cost-of-trycatch)
    - [Function Try Blocks](#function-try-blocks)
    - [Function Try Block for a Constructor](#function-try-block-for-a-constructor)
    - [New in C++11](#new-in-c11)

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

### **Guideline-1**

- Throw by value
- Catch by reference

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
