# Concepts

> [C++20新特性之concept](https://uint128.com/2020/11/20/C-20%E6%96%B0%E7%89%B9%E6%80%A7%E4%B9%8Bconcept/)

指对一个模板参数的具体说明，依赖`Concepts`，编译器可以在模板实例化之前对参数进行检查，并得到更加清晰的错误提示。C++20之前，使用`enable_if`或者`实例化失败`可以达到类似的目的，但生成的错误信息晦涩难懂。`Concepts`会更早失败并提供清晰的错误提示。

**`concept`的本质是一个模板的编译期的`bool`变量**

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

## Concept

`Concept`是一个有名的集合，包含一组约束或这些约束的逻辑组合。`Concept`和`require表达式`都被翻译为编译器的`bool`变量，可以当做普通变量来使用，比如 `if constexpr`

```cpp
template<typename T>
concept Addable = requires(T a, T b)
{
    a + b;
};

template<typename T>
concept Dividable = requires(T a, T b)
{
    a/b;
};

template<typename T>
concept DivAddable = Addable<T> && Dividable<T>;

template<typename T>
void f(T x)
{
    if constexpr(Addable<T>){ /*...*/ }
    else if constexpr(requires(T a, T b) { a + b; }){ /*...*/ }
}
```

## some example

```cpp
// 一个永远都能匹配成功的concept
template <typename T>
concept always_satisfied = true; 

// 一个约束T只能是整数类型的concept，整数类型包括 char, unsigned char, short, ushort, int, unsinged int, long等。
template <typename T>
concept integral = std::is_integral_v<T>;

// 一个约束T只能是整数类型，并且是有符号的concept
template <typename T>
concept signed_integral = integral<T> && std::is_signed_v<T>;
```

如何使用上述定义的concept

```cpp
// 任意类型都能匹配成功的约束，因此mul只要支持乘法运算符的类型都可以匹配成功。
template <always_satisfied T>
T mul(T a, T b) {
    return a * b;
}

// 整型才能匹配add函数的T
template <integral T>
T add(T a, T b) {
    return a + b;
}

// 有符号整型才能匹配subtract函数的T
template <signed_integral T>
T subtract(T a, T b) {
    return a - b;
}

int main() {
    mul(1, 2); // 匹配成功, T => int
    mul(1.0f, 2.0f);  // 匹配成功，T => float

    add(1, -2);  // 匹配成功, T => int
    add(1.0f, 2.0f); // 匹配失败, T => float，而T必须是整型
    subtract(1U, 2U); // 匹配失败，T => unsigned int,而T必须是有符号整型
    subtract(1, 2); // 匹配成功, T => int
}
```

为了满足不同喜好的提案，C++标准委员会支持了**3种**`concept`的用法，同时还支持其在lambda表达式中的使用，*有点脱离群众*：

```cpp
// 约束函数模板方法1
template <my_concept T>
void f(T v);

// 约束函数模板方法2
template <typename T>
requires my_concept<T>
void f(T v);

// 约束函数模板方法3
template <typename T>
void f(T v) requires my_concept<T>;

// 直接约束C++14的auto的函数参数
void f(my_concept auto v);

// 约束模板的auto参数
template <my_concept auto v>
void g();

// 约束auto变量
my_concept auto foo = ...;

// 约束lambda函数的方法1
auto f = []<my_concept T> (T v) {
  // ...
};
// 约束lambda函数的方法2
auto f = []<typename T> requires my_concept<T> (T v) {
  // ...
};
// 约束lambda函数的方法3
auto f = []<typename T> (T v) requires my_concept<T> {
  // ...
};
// auto函数参数约束
auto f = [](my_concept auto v) {
  // ...
};
// auto模板参数约束
auto g = []<my_concept auto v> () {
  // ...
};
```