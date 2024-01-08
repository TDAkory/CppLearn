# Type Programming in C++

- Type is a set of values

- c++支持在函数调用时发生有限的隐式类型转换

```cpp
int add(int a, int b);

// All the following codes can be compiled!
int a = 0; int b = 1; add(a, b);
long a = 0; long b = 1; add(a, b);
float a = 0; double b = 1; add(a, b);
```

- 可以使用模板元编程来防止隐式类型转换