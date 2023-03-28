# [A New Way of Meta-Programming? - Kris Jusiak - CppCon 2022](https://www.youtube.com/watch?v=zRYlQGMdISI)

## 1. Design by introspection

```cpp
consteval auto foo(auto t) {
    if constexpr(requires{ t.foo; }) {
        return t.foo;
    } else {
        return 0;
    }
}

constexpr struct { int foo{42}; } f;
static_assert(42 == foo(F));

constexpr struct { int bar{42}; } b;
static_assert(0 == foo(b));
```

## 2. Immediately-Invoked Function Expression

```cpp
template<auto N>
constexpr auto unrool = [](auto expr) {
    [expr]<auto ...Is>(std::index_sequence<Is...>) {
        ((expr(), void(Is)), ...);
    }(std::make_index_sequence<N>{});
};
```

## 3. Fixed String

## 4. Reflection to_tuple

## 5. constexpr std::vector