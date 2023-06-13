# [Iterator](https://cplusplus.com/reference/iterator/)

## 定义自己的迭代器

在C++17之前，实现自定义迭代器推荐从std::iterator中派生

```cpp
template<
    class Category,
    class T,
    class Distance = std::ptrdiff_t,
    class Pointer = T*,
    class Reference = T&
> struct iterator;
```

`T`是你的容器类类型，无需多提。而 `Category` 是必须首先指定的[迭代器标签](https://en.cppreference.com/w/cpp/iterator/iterator_tags)
