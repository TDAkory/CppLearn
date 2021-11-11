# Concepts

指对一个模板参数的具体说明，依赖`Concepts`，编译器可以在模板实例化之前对参数进行检查，并得到更加清晰的错误提示。C++20之前，使用`enable_if`或者`实例化失败`可以达到类似的目的，但生成的错误信息晦涩难懂。`Concepts`会更早失败并提供清晰的错误提示。

## Requires expression

`requires`是关键字，用来开始一个`Requires expression`，是一个类型为`bool`的右值表达式。

`Requires expression`是一个包含模板参数限制的表达式，当实例化参数满足限制时，表达式返回`true`，

```cpp
template<typename T> /*...*/
requires (T x) // optional set of fictional parameter(s)
{
    // simple requirement: expression must be valid
    x++;    // expression must be valid
    
    // type requirement: `typename T`, T type must be a valid type
    typename T::value_type;
    typename S<T>;

    // compound requirement: {expression}[noexcept][-> Concept];
    // {expression} -> Concept<A1, A2, ...> is equivalent to
    // requires Concept<decltype((expression)), A1, A2, ...>
    {*x};  // dereference must be valid
    {*x} noexcept;  // dereference must be noexcept
    // dereference must  return T::value_type
    {*x} noexcept -> std::same_as<typename T::value_type>;
    
    // nested requirement: requires ConceptName<...>;
    requires Addable<T>; // constraint Addable<T> must be satisfied
};
```
