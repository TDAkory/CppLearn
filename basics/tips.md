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