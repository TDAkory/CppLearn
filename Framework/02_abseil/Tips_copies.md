
## 核心原则：“一个名称，无拷贝；两个名称，两次拷贝”

1. **基本规则**：在评估一个作用域内是否会发生拷贝时，关键在于检查数据有多少个**活跃的名称（变量名）**。当同一份数据同时存在两个独立的名称时，编译器就（且通常必须）进行一次拷贝。反之，如果只有一个名称，编译器在绝大多数情况下能够优化掉（elide）拷贝。

2. **现代C++的保证**：得益于C++11引入的**移动语义**（STL容器会自动使用）和编译器的**拷贝构造函数省略** 技术，上述规则不仅是一个下限，在很多时候几乎成为一个**保证**。如果性能分析显示有超出此规则的拷贝，那很可能是编译器bug。

## 示例

通过避免引入不必要的额外名称，可以帮助编译器消除拷贝

```cpp
std::string build();

std::string foo(std::string arg) {
  return arg;  // no copying here, only one name for the data “arg”.
}

void bar() {
  std::string local = build();  // only 1 instance -- only 1 name

  // no copying, a reference won’t incur a copy
  std::string& local_ref = local;

  // one copy operation, there are now two named collections of data.
  std::string second = foo(local);
}
```

### 最重要的建议

文档最后强调，**代码的可读性和一致性远比纠结于拷贝重要**。正确的做法是：

1. **先分析（Profile），后优化**：不要盲目优化，应先通过性能分析工具定位瓶颈。
2. **拥抱现代C++**：不要固守十年前对C++拷贝性能的陈旧认知。现代C++允许你编写返回值的清晰API，而无需过分担心性能开销，因为编译器优化远比想象中强大。

## Ref

- [Tip of the Week #24: Copies, Abbrv.](https://abseil.io/tips/24)