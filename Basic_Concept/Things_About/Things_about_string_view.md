# string_view

## 一些建议

* 使用string_view时应当像primitive type一样用值传递，比如int、double

* 将 string_view 标记为 const 只影响 string_view 对象本身是否可以被修改，而不是它是否可以用来修改底层字符——它永远不能
  
* string_view 不一定是以 NUL 结尾的。因此，写以下代码是不安全的：
  
```cpp
printf("%s\n", sv.data()); // 不要这样做！

absl::PrintF("%s\n", sv);
```

* 你可以像记录一个字符串或 const char* 一样记录一个 string_view：

```cpp
LOG(INFO) << "Took '" << sv << "'";
```

* string_view has a constexpr constructor and a trivial destructor; keep this in mind when using in static and global variables

* string_view 是一种有效的引用，可能不是成员变量的最佳选择。因为生命周期的关系，string_view的生命周期不应比其底层实际字符串的生命周期更长

## Ref

*[Tip of the Week #1: string_view](https://abseil.io/tips/1)