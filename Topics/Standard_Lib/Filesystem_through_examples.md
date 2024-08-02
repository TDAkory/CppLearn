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

`filesystem`在错误处理上，兼容了两种风格：即支持抛出异常，也通过函数重载支持返回错误码。

## 操作文件夹

### 创建 [create_directory](https://en.cppreference.com/w/cpp/filesystem/create_directory)

`create_directory`用于创建一级文件夹，即要求其父路径必须是存在的。如果文件夹是存在的，不会报错。

```cpp
// 会抛出异常
bool create_directory(const path& __p);
// 会返回错误码
bool create_directory(const path& __p, error_code& __ec) noexcept;
```

```cpp
// 示例
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

其重载形式额外支持同步目标文件的权限，`existing_p`必须是一个存在的文件夹。

```cpp
bool create_directory(const std::filesystem::path& p,
                      const std::filesystem::path& existing_p );

bool create_directory(const std::filesystem::path& p,
                      const std::filesystem::path& existing_p,
                      std::error_code& ec ) noexcept;
```

具体什么权限被拷贝，取决于操作系统的实现。在Posix系统上，其行为类比如下：

```shell
stat(existing_p.c_str(), &attributes_stat)
mkdir(p.c_str(), attributes_stat.st_mode)
```

此外还支持创建多级文件夹，通过接口`create_directories`：

```cpp
bool create_directories( const std::filesystem::path& p );

bool create_directories( const std::filesystem::path& p, std::error_code& ec );
```

```cpp
// 示例
#include <filesystem>
#include <exception>

{
    std::filesystem::path nested = "a/b/c";
    try {
        if (std::filesystem::create_directories(nested))
            // do something
        else
            // do some other thing
    }
    catch (const std::exception& ex) {
        // do exception handling
    }
}
```

### 删除

```cpp
bool remove( const std::filesystem::path& p );

bool remove( const std::filesystem::path& p, std::error_code& ec ) noexcept;
```

删除文件或空的文件夹。可以删除符号链接，不会删除链接的目标。文件删除返回`true`、文件不存在返回`false`

```cpp
// 示例
#include <filesystem>
#include <exception>

int main() {
    std::filesystem::path dir = "test";
    try {
        if (std::filesystem::create_directory(dir))
            // do something
        else
            // do some other thing

        if (std::filesystem::remove(dir))
            // do something
        else
            // do some other thing
    }
    catch (const std::exception& ex) {
        // do exception handling
    }
}
```

```cpp
std::uintmax_t remove_all( const std::filesystem::path& p );
// LWG 3014：C++17中remove_all的error_code重载错误地标记为noexcept，但实际上可能会分配内存。因此，noexcept被移除。
std::uintmax_t remove_all( const std::filesystem::path& p, std::error_code& ec );
```

递归删除由路径p指定的目录及其所有子目录和内容，然后删除p本身。返回删除的文件和目录的数量。

如果底层操作系统API出现错误，remove和remove_all可能会抛出std::filesystem::filesystem_error。

```cpp
// 示例
#include <filesystem>
#include <exception>

int main() {
    std::filesystem::path dir = "test";
    std::filesystem::path nested = dir / "a/b";
    std::filesystem::path more = dir / "x/y";
    try {
        if (std::filesystem::create_directories(nested) &&
            std::filesystem::create_directories(more))
            // do something
        else
            // do some other thing

        const auto cnt = std::filesystem::remove_all(dir);
    }
    catch (const std::exception& ex) {
        // do exception handling
    }
}
```

### 遍历

通过[`directory_iterator`](https://en.cppreference.com/w/cpp/filesystem/directory_iterator)可以很方便的完成文件夹的遍历，它会遍历`directory_entry`对象，但不会递归遍历子文件夹，遍历的顺序是随机的，每个`directory_entry`对象只访问一次。特殊的路径名（`.`, `..`）将被跳过。

迭代器的表现和一般容器迭代的表现类似：

* 当遍历结束时，迭代器会自动转换为`end()`，在`end()`上自增是未定义行为
* 如果在创建迭代器之后，文件夹中的文件发送变化（子文件、子文件夹的创建和删除），迭代器的行为是未定义的（可能感知变化、可能不感知）

```cpp
// 示例
#include <filesystem>
#include <iostream>

void ls() {
    for (const auto& entry : std::filesystem::directory_iterator(".")) 
        std::cout << entry.path() << '\n';
}
```

上述迭代器不支持递归扫描，标准库提供了`recursive_directory_iterator`，它会递归遍历子文件夹。

```cpp
// 示例
#include <filesystem>
#include <iostream>

void ls() {
    for (const auto& entry : std::filesystem::recursive_directory_iterator(".")) 
        std::cout << entry.path() << '\n';
}
```

### 临时文件夹

`filesystem`还提供了接口来返回一个临时文件夹，用来存放临时的文件。在`Posix`文件系统上，临时文件的路径可以通过环境变量`TMPDIR`, `TMP`, `TEMP`, `TEMPDIR`设置，或返回`/tmp`。在`Windows`系统上，临时文件的路径通常是`GetTempPath`的返回值。

```cpp
path temp_directory_path();
path temp_directory_path( std::error_code& ec );
```

```cpp
// 示例
#include <filesystem>
#include <iostream>
namespace fs = std::filesystem;
 
int main()
{
    std::cout << "Temp directory is " << fs::temp_directory_path() << '\n';
}

// Possible Output: Temp directory is "C:\Windows\TEMP\"
```

## 操作文件

### 拷贝

在拷贝的语义上，`filesystem`提供了三个主要的函数，分别是：`copy`，`copy_file`，`copy_symlink`

* [std::filesystem::copy](https://en.cppreference.com/w/cpp/filesystem/copy)：拷贝文件或文件夹

```cpp
void copy(const std::filesystem::path& from,
          const std::filesystem::path& to );

void copy(const std::filesystem::path& from,
          const std::filesystem::path& to,
          std::filesystem::copy_options options,
          std::error_code& ec );
```

```cpp
// 示例
#include <filesystem>

int main() {
    std::filesystem::path src = "source_file.txt";
    std::filesystem::path dest = "destination_file.txt";
    try {
        std::filesystem::copy(src, dest);
    } catch (std::filesystem::filesystem_error& e) {
        // do exception handling
    }
}
```

如果想要递归的拷贝文件夹，则可以使用[copy_options](https://en.cppreference.com/w/cpp/filesystem/copy_options)来支持定制化拷贝执行

```cpp
#include <filesystem>
#include <fstream>

void create_temp_directories_and_files() {
    std::filesystem::create_directories("source_directory/subdir1");
    std::filesystem::create_directories("source_directory/subdir2");

    std::ofstream("source_directory/file1.txt") << "This is file 1";
    std::ofstream("source_directory/subdir1/file2.txt") << "This is file 2";
    std::ofstream("source_directory/subdir2/file3.txt") << "This is file 3";
}

int main() {
    create_temp_directories_and_files();

    std::filesystem::path src = "source_directory";
    std::filesystem::path dest = "destination_directory";
    try {
        std::filesystem::copy(src, dest, std::filesystem::copy_options::recursive);
        // do something
    } catch (std::filesystem::filesystem_error& e) {
        // do exception handling
    }

    for (const auto& entry : std::filesystem::recursive_directory_iterator(dest)) {
        // do something with entry
    }
}
```

* [std::filesystem::copy_file](https://en.cppreference.com/w/cpp/filesystem/copy_file)：拷贝文件
* [std::filesystem::copy_symlink](https://en.cppreference.com/w/cpp/filesystem/copy_symlink)：拷贝符号链接

### 移动和文件重命名

[std::filesystem::rename](https://en.cppreference.com/w/cpp/filesystem/rename)

```cpp
// 示例
#include <filesystem>
#include <fstream>

{
    std::ofstream("old_file.txt") << "This is file 1";
    std::filesystem::path old_name = "old_file.txt";
    std::filesystem::path new_name = "new_file.txt";
    try {
        std::filesystem::rename(old_name, new_name);
        // do something
    } catch (std::filesystem::filesystem_error& e) {
        // do exception handling
    }
}
```

### 创建链接

[`create_hard_link`](https://en.cppreference.com/w/cpp/filesystem/create_hard_link)：创建硬链接

[`create_symlink`、`create_directory_symlink`](https://en.cppreference.com/w/cpp/filesystem/create_symlink)：创建软链接
  
```cpp
// 示例
#include <filesystem>
#include <fstream>
#include <format>

{
    std::ofstream("target_file.txt") << "This is file 1";
    std::filesystem::path target = "target_file.txt";
    std::filesystem::path link = "hard_link_file.txt";
    try {
        std::filesystem::create_hard_link(target, link);
        // do something
    } catch (std::filesystem::filesystem_error& e) {
        // do exception handling
    }
}
```

下面的例子可以更好的理解硬链接和软链接的区别：

```cpp
// 示例
#include <filesystem>
#include <iostream>
#include <fstream>

void display_file_content(const std::filesystem::path& path) {
    if (std::filesystem::exists(path)) {
        std::ifstream file(path);
        std::string content((std::istreambuf_iterator<char>(file)), std::istreambuf_iterator<char>());
        std::cout << "Content of " << path << ": " << content << '\n';
    } else {
        std::cout << path << " does not exist.\n";
    }
}

int main() {
    std::filesystem::path original_file = "original_file.txt";
    std::filesystem::path symlink = "symlink_to_file.txt";
    std::filesystem::path hardlink = "hardlink_to_file.txt";

    // Step 1: Create the original file
    std::ofstream(original_file) << "Hello World!";
    std::cout << "Original file created.\n";
    display_file_content(original_file);

    // Step 2: Create a symbolic link to the original file
    try {
        std::filesystem::create_symlink(original_file, symlink);
        std::cout << "Symbolic link created successfully.\n";
        display_file_content(symlink);
    } catch (std::filesystem::filesystem_error& e) {
        std::cout << e.what() << '\n';
    }

    // Step 3: Create a hard link to the original file
    try {
        std::filesystem::create_hard_link(original_file, hardlink);
        std::cout << "Hard link created successfully.\n";
        display_file_content(hardlink);
    } catch (std::filesystem::filesystem_error& e) {
        std::cout << e.what() << '\n';
    }

    // Step 4: Delete the original file and compare...
    std::filesystem::remove(original_file);
    std::cout << "Original file deleted.\n";
    display_file_content(symlink);
    display_file_content(hardlink);
}
```

## 操作文件路径

### 检查存在性

通过[std::filesystem::exists](https://en.cppreference.com/w/cpp/filesystem/exists)检查文件和文件夹的存在性

```cpp
//
#include <filesystem>

{
    std::filesystem::path p = "example_file.txt";
    if (std::filesystem::exists(p))
        // do something
    else
        // do some other thing
}
```

### 判断是否是文件夹

[std::filesystem::is_regular_file](https://en.cppreference.com/w/cpp/filesystem/is_regular_file)用来识别文件，[std::filesystem::is_directory](https://en.cppreference.com/w/cpp/filesystem/is_directory)用来识别文件夹

```cpp
// 示例
#include <filesystem>

{
    std::filesystem::path p = "example_path";
    if (std::filesystem::is_regular_file(p))
        // do something
    else if (std::filesystem::is_directory(p))
        // do something
    else
        // do some other thing
}
```

除了这两个接口之外，`filesystem`还提供了一些其他的工具函数，判断文件是否是链接、是否是socket等，他们的用法都是类似的。

### 读取链接文件的状态

[std::filesystem::read_symlink](https://en.cppreference.com/w/cpp/filesystem/read_symlink)

```cpp
std::filesystem::path read_symlink( const std::filesystem::path& p );
```

## 其他操作

## Refs

- [22 Common Filesystem Tasks in C++20](https://www.cppstories.com/2024/common-filesystem-cpp20/)
- [Displaying File Time in C++: Finally fixed in C++20](https://www.cppstories.com/2024/file-time-cpp20/)
- [C++17 in Detail: Filesystem in The Standard Library](https://www.cppstories.com/2017/08/cpp17-details-filesystem/)