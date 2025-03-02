# Copy

## `One Name, No Copy; Two Names, Two Copies`

当评估任何作用域内是否会发生数据拷贝（包括触发返回值优化 RVO 的情况）时，检查你的数据引用了多少个名字。
在任何有两个名字同时指向数据的地方，你都会有两个数据的拷贝。大致上说，编译器会在所有其他情况下（并且通常必须）省略拷贝操作。

```cpp
// One Name, No Copy
std::string createString() {
    std::string s = "Hello, World!";
    return s;  // Only one name: 's'. Compiler can elide the copy.
}

int main() {
    std::string result = createString();  // No copy, RVO applies.
    std::cout << result << std::endl;
    return 0;
}
```

```cpp
// Two Names, Two Copies
std::string createString() {
    std::string s1 = "Hello, World!";
    std::string s2 = s1;  // Two names: 's1' and 's2'. A copy is made.
    return s2;  // Another copy may be made unless RVO applies.
}

int main() {
    std::string result = createString();  // RVO may apply here.
    std::cout << result << std::endl;
    return 0;
}
```

```cpp
// Move Semantics
std::string createString() {
    std::string s1 = "Hello, World!";
    std::string s2 = std::move(s1);  // Two names, but move semantics avoid a copy.
    return s2;  // RVO may apply here.
}

int main() {
    std::string result = createString();  // No copy, RVO applies.
    std::cout << result << std::endl;
    return 0;
}
```

1. **避免不必要的命名**：
   - 为了最小化拷贝，避免引入不必要的名称（变量）来引用相同的数据。
   - 例如，优先直接返回值，而不是将其赋值给中间变量。

2. **使用移动语义**：
   - 当需要转移资源的所有权时，使用 `std::move` 来避免拷贝。

3. **信任编译器**：
   - 现代 C++ 编译器非常擅长优化掉不必要的拷贝，尤其是在 RVO 和移动语义的情况下。
   - 如果你怀疑有不必要的拷贝发生，检查代码中是否有额外的名称或遗漏的 `std::move` 调用。

4. **基准测试和分析**：
   - 如果基准测试显示的拷贝次数超出预期，可能是编译器 bug 或代码结构问题。
   - 使用工具如 `valgrind` 或编译器特定的标志来分析拷贝和移动行为。