# [UB]()

- [Undef and Poison: Present and Future](chrome-extension://ikhdkkncnoglghljlkmcimlnlhkeamad/pdf-viewer/web/viewer.html?file=https%3A%2F%2Fllvm.org%2Fdevmtg%2F2020-09%2Fslides%2FLee-UndefPoison.pdf#=&zoom=page-width)

## 从一个死循环说起

```cpp
__attribute__((noinline))
void bar(unsigned int &count) {
  count++;
}
__attribute__((noinline))
bool foo(int x) {
  unsigned count = 0;
  for (unsigned i = 0; i < x; i++ ) {
    bar(count);
  }
}
int main() {
  foo(26);
}
```

上述代码存在什么问题，初看是会执行26次无意义的循环。但实际上会陷入死循环。

[As of r165273, clang produces 'unreachable' if we flow off the end of a value-returning function (without returning a value) in C++. This is undefined behavior, but we used to just return undef, instead of trying to exploit it.](https://lists.llvm.org/pipermail/cfe-dev/2012-October/024730.html)

## What is

1. [ill-formed](http://eel.is/c++draft/defns.ill.formed)- program that it not [well-formed](http://eel.is/c++draft/defns.well.formed)

2. ill-formed [no diagnostic required](https://en.cppreference.com/w/cpp/language/ndr) - IFNDR

3. [implementation-defined behavior](http://eel.is/c++draft/defns.impl.defined) - Number of bits in a byte. see [intro.memory](http://eel.is/c++draft/intro.memory) 

4. [unspecified behavior](http://eel.is/c++draft/defns.unspecified) - How the memory for an exception object is allocated. See [except.throw](http://eel.is/c++draft/except.throw#4)

5. [undefined behavior](http://eel.is/c++draft/defns.undefined) - like signed integer overflow. see [expr](http://eel.is/c++draft/expr#pre-4)

UB放宽了约束，增加了编译器可以自由发挥的空间。

## UB lets compiler do anything!

现实一：compiler 会尽量在 frontend 通过 warning 的方式发现 UB 相关的问题。囿于前端较难进行数据流分析，从而难以发现深层次的bug

现实二：虽然 backend 可以做强大的数据流分析，但是llvm ir丢失了高层次语言相关的信息，给出有效的 UB warning 力有不逮

现实三：提供了Undefined Behavior Sanitizer 等工具来协助发现问题

现实四："Undefined Beahvior lets compiler do anything"。既然无法反抗，和不闭上眼睛，进行极致的优化呢。

## How to avoid

> 利用编译器来揪出UB

### compiler warnings

* -Wreturn-type
* -Wno-non-pod-varargs
* -Wcall-to-pure-virtual-from-ctor-dtor
* -Wdelete-incomplete-Wformat=2
* -Wnull-dereference
* -Wnull-pointer-arithmetic
* -Wpointer-arith
* -Warray-bounds-pointer-arithmetic
* -Wundefined-reinterpret-cast

### 降低优化等级

### -ftrapv

### Undefined Behavior Sanitizer

### -fno-strict-aliasing

### -fno-delete-null-pointer-checks

### -fno-strick-overflow

### -fno-strict-return

### -ftrivial-auto-var-init=zero