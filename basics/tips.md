# Tips

### 1. 尽量不要用全局变量， 编译器很难对全局做优化！
```cpp
// one file
const char *str = "HELLO ByteDance";

char foo() {
        return str[2];
}

// another file
extern const char *str;  // 变量类型不属于签名的一部分
struct A {
        A() { str = "WORLD"; }
};

A a;
```
```Assembly
_Z3foov:                             # @_Z3foov
        movq    str(%rip), %rax      # 读取str地址
        movb    2(%rax), %al         # 读取str[2] 内容
        retq
```
foo() 的结果就不再是'L', 所以编译器是不能做这个优化的。

### 2. 如果一定要用全局变量，加上static修饰
static修饰之后，编译器可以确定str不会被其他编译单元访问，就可以做很多静态分析。
```cpp
static const char *str = "HELLO ByteDance";

char foo() {
        return str[2];
}
```
```Assembly
_Z3foov:                                # @_Z3foov
        movb    $76, %al                # 76 是'L'的ASCII
        retq
```

### 3. std::string vs std::string_view
- std::string_view 本质上就是一个const char*的指针，所以字符串常量的初始化建议用std::string_view. 
- std::string_view 可以转成std::string

### 4. 不要在头文件中定义容器对象
```cpp
#include 

static const std::string MODEL_X = "MODEL_X";
static const std::string MODEL_Y = "MODEL_Y";
```
`MODEL_X` 和`MODEL_Y` 都会被创建出来，不管cpp里面有没有用到。而且任何include "xxx.h" 所编译出来的.o 文件里面都会创建这些对象。也就是链接出来的二进制中有很多份`MODEL_X`和`MODEL_Y`.

基于上述理由，在头文件定义表也是不优雅的，可以通过一个def文件来描述
```cpp
#ifndef MAP
#define MAP(x, y)
#endif

MAP(HELLO, 4)
MAP(WORLD, 6)

#undef MAP
```

```cpp
const std::unordered_map<std::string, int> map = {
#define MAP(x, y) {#x, y},
#include "xxx.def"
};

// 等同于
static const std::unordered_map<std::string, int> map = {
    {"HELLO", 4}, 
    {"WORLD", 6}
};
```

### 5. 多用`std::array`而不是`std::vector`
```cpp
#include <vector>
#include <string_view>

static const std::vector vec = {"HELLO", "ByteDance"};
std::string_view foo() {
        return vec[0];
}
```

vec对于编译器来说是一个变量，所以在构造string_view 的时候，会调用strlen去计算长度。而且编译器会把整个std::vector 构建出来。如果换成std::array, vec 是一个常量，编译器知道vec[0] 就是“HELLO”。最终出来的汇编如下：
```shell
_Z3foov: # @_Z3foov
    movl $5, %eax
    movl $.L.str, %edx # .L.str 是在str段的位置
    retq
```
std::array vec = {"HELLO"} 这种在构造函数自动推到模版类型的写法是在C++17之后支持的。

由于这种constexpr容器对性能确实有很大的提升，C++20 开始支持更多的constexpr容器。[More constexpr containers](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0784r7.html)

### 6.尽量避免指针转换
-   strict-aliasing
-   不要试图对cast过的指针进行解引用，优化开启的时候，有可能行为会达不到预期。完全取决于编译器实现。
-   如果发现优化开起来程序行为出问题，试试选项 -fno-strict-aliasing
-   关掉strict-aliasing会对性能有巨大影响

### 7.慎用C++异常
C++异常的主要特点是

-   对于不抛异常的这条路，是零开销的
-   一旦抛异常，运行库会做两次调用栈回溯，这个过程是极其耗费时间的

所以，异常被设计出来做极端情况的容错处理，而不是用来处理代码逻辑的。

### 8. `auto` vs `auto &`
只有当你需要一份值拷贝的时候，用 `auto`，不然如果是指针，用`auto *`, 如果是值，用`auto &`.

### 9. 短函数尽量在头文件实现