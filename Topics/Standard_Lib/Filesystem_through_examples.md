# Filesystem through examples

> [Filesystem library (since C++17)](https://en.cppreference.com/w/cpp/filesystem)

## 一些基本信息

在C++ 17之前，C++/C提供了两种文件操作的机制。一种是C++标准库定义的文件流fstream，另一种是C标准库定义的文件操作类型FILE。fstream 更好地集成了C++的特性，如异常处理和类型安全，适合需要C++特性的项目，而 FILE 更适合底层或性能敏感的应用，以及需要与C代码兼容的场景。

尽管fstream提供了针对文件的操作流，但其仍然存在一些问题。比如与C语言的 FILE* 流相比，fstream 可能在某些情况下性能较低，尤其是在需要大量I/O操作的场景中；fstream无法完全屏蔽不同操作系统在文件和路径表示上的差异（分隔符、长度和字符集限制、权限模型、结束符等）

因此，C++17引入了<filesystem>库，这是C++标准中首个专门用于文件系统操作的库。与共享指针、正则表达式一样，
filesystem也是由boost.filesystem发展来的，其最终提案是[P0218R0: Adopt the File System TS for C++17](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0317r1.html)

// TODO 核心对象

## Refs

- [22 Common Filesystem Tasks in C++20](https://www.cppstories.com/2024/common-filesystem-cpp20/)
- [Displaying File Time in C++: Finally fixed in C++20](https://www.cppstories.com/2024/file-time-cpp20/)
- [C++17 in Detail: Filesystem in The Standard Library](https://www.cppstories.com/2017/08/cpp17-details-filesystem/)