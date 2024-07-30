# Filesystem through examples

> [Filesystem library (since C++17)](https://en.cppreference.com/w/cpp/filesystem)
> [Filesystem source code of libstdc++](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/std/filesystem)

## 一些基本信息

在C++ 17之前，C++/C提供了两种文件操作的机制。一种是C++标准库定义的文件流fstream，另一种是C标准库定义的文件操作类型FILE。fstream 更好地集成了C++的特性，如异常处理和类型安全，适合需要C++特性的项目，而 FILE 更适合底层或性能敏感的应用，以及需要与C代码兼容的场景。

尽管fstream提供了针对文件的操作流，但其仍然存在一些问题。比如与C语言的 FILE* 流相比，fstream 可能在某些情况下性能较低，尤其是在需要大量I/O操作的场景中；fstream无法完全屏蔽不同操作系统在文件和路径表示上的差异（分隔符、长度和字符集限制、权限模型、结束符等）

因此，C++17引入了<filesystem>库，这是C++标准中首个专门用于文件系统操作的库。与共享指针、正则表达式一样，
filesystem也是由boost.filesystem发展来的，其最终提案是[P0218R0: Adopt the File System TS for C++17](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0317r1.html)

filesystem定义了一些核心类型：

* `file`：文件对象持有文件的句柄，可以读写数据，包含名称、参数、状态等信息，可以是目录、普通文件、符号链接等
* [`path`](https://en.cppreference.com/w/cpp/filesystem/path)
  * path对象可以隐式转换为std::wstring或std::string。这意味着你可以直接将path对象传递给需要字符串的文件流函数
  * 可以从std::string、const char*、string_view 等字符串类型初始化path对象
  * 提供了begin()和end()成员函数，使其可以像容器一样被迭代。这允许你遍历路径中的每个组成部分
  * 处理了不同操作系统间的路径表示差异，提供了跨平台的文件路径操作
* [`directory_entry`](https://en.cppreference.com/w/cpp/filesystem/directory_entry)

## 操作文件夹

### 创建 [create_directory](https://en.cppreference.com/w/cpp/filesystem/create_directory)

```cpp
// 会抛出异常
bool create_directory(const path& __p);
// 会返回错误码
bool create_directory(const path& __p, error_code& __ec) noexcept;
```

```cpp
#include <filesystem>

{
    // example 1
    std::filesystem::path dir = "the_path_of_dir";
    if (std::filesystem::create_directory(dir)) {
        // do something
    } else {
        // do some other thing
    }
}

{
    // example 2
    std::filesystem::path dir = "the_path_of_dir";
    std::error_code ec{};
    if (std::filesystem::create_directory(dir, ec)) {
        // do something
    } else {
        // do some other thing, `ec.message()` returns a string
    }
}

```

## 操作文件

## 操作文件路径

## 检查存在性

## 其他操作

## Refs

- [22 Common Filesystem Tasks in C++20](https://www.cppstories.com/2024/common-filesystem-cpp20/)
- [Displaying File Time in C++: Finally fixed in C++20](https://www.cppstories.com/2024/file-time-cpp20/)
- [C++17 in Detail: Filesystem in The Standard Library](https://www.cppstories.com/2017/08/cpp17-details-filesystem/)