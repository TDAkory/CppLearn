## Attentions

* 模板不允许自动类型转换


## 重载函数模板

```cpp
inline int const& max(int const& a, int const& b) {
    return a < b ? b : a;
}

template <typename T>
inline T const& max(T const& a, T const& b) {
    return a < b ? b : a;
}

template <typename T>
inline T const & max(T const& a, T const& b, T const& c) {
    return ::max(::max(a, b), c);
}

int main () {
    ::max(1, 2, 3);     // 调用三参数的模板
    ::max(2.0, 32.0);   // max<double> 实参推断
    ::max('a', 'c');    // max<char> 实参推断
    ::max(5, 98);       // 调用int重载的非模板函数
    ::max<>(7, 42);     // 调用max<int> 实参推断
    ::max<double>(1, 2.0);  // 调用max<double> 显式指定，未发生实参推断
    ::max('a', 42.7);       // 调用int重载的非模板函数
}
```
如上，一个非模板函数可以和一个同名的函数模板同时存在，且该函数模板可以被实例化为这个非模板函数。重载解析的过程中，相同条件下，非模板函数优先级高；否则，将选择更匹配的模板实例化。
