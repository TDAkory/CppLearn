# [What volatile means and doesn't means in cpp](https://www.youtube.com/watch?v=GeblxEQIPFM&list=PLHTh1InhhwT6U7t1yP2K8AtTEKmcM3XU_&index=2)

- The **volatile qualifier** is a vital tool for preventing compilers from performing certain harmful optimizations.

- Unfortunately, many C++ programmers aren’t clear on exactly what protections volatile provides.

- As such, many programmers apply the volatile qualifier incorrectly.

- A misapplied volatile might:
  - prevent optimizations unnecessarily, or worse
  - fail to provide the expected protection, leading to subtle run-time bugs.

> It means that the object is not only accessed by C++ code.
> 
> For example, if you have a memory mapped device, the device can read and modify the mapped memory region. So you need to use the volatile keyword to tell the compiler that these regions of memory are not only accessed by C++ code, but also by the device.
> 
> A volatile member function is a member function that can be used on volatile objects.

```cpp
std::uint32_t &USTAT0 = *reinterpret_cast<special_register *>(0x03FFD008);
std::uint32_t &UTXBUF0 = *reinterpret_cast<special_register *>(0x03FFD00C);

while ((USTAT0 & TBE) == 0) {
}
UTXBUF0 = c;
```

尽管这段代码看起来应该可以正常工作，但编译器的优化器可能会导致这段代码失败。这是因为编译器可能会重新排序指令以提高效率，这可能会破坏对硬件寄存器的时序要求

```cpp
while ((USTAT0 & TBE) == 0) {
}
UTXBUF0='\r';
while ((USTAT0 & TBE) == 0) {
}
UTXBUF0='\n';
```

编译器视角，循环体中没有操作会改变循环条件，可能会将上述代码优化到如下形式：永远循环 or 跳过循环

```cpp
if ((USTAT0 & TBE) == 0) {
    for (;;) {
    }
}
```

结合到前面的示例，会发现两次判断是相同的，则可能进一步被优化到如下形式：

```cpp
if ((USTAT0 & TBE) == 0) {
    for (;;) {
    }
}
UTXBUF0='\r';
UTXBUF0='\n';   // second if-statement removed
```

然后编译器又发现对同一个位置连续两次赋值，中间没有任何操作，可能进一步优化到如下形式：

```cpp
if ((USTAT0 & TBE) == 0) {
    for (;;) {
    }
}
UTXBUF0='\n';
```

1. use toolchains ability to disable all optimizations for a region of code, however which will be overkill
2. volatile can provide a more precise solution

**`const` and `volatile` are the only symbols(in C++) that can appear either as declaration specifiers or in declarators**

* for inter-thread communications, use synchronization tools such as mutexes and semaphores
* don't use `volatile` objects for inter-thread communications
* accesses to volatile objects are not guaranteed to be atomic

if we suspect that your compiler is mishandling a volatile object, we have a few options:

* turn off optimizations for the affected code
* use a different version of compiler
* use a workaround suggested by Eide and Regehr

```cpp
int vol_read_int(int volatile &vp) {
    return vp;
}

int volatile v_int;

int value = vol_read_int(v_int);

int volatile &vol_write_int(int volatile &v) {
    return v;
}

int volatile v_int_2;

vol_write_int(v_int_2) = 256;
```

上述workaround为啥能起作用：

* 编译器编译对非内联函数（ non - inline function ）的调用时，不清楚函数会执行什么操作、有何种副作用
* 为保证正确性，编译器必须严格按用户期望的次数去调用函数
* 这种逻辑和访问 volatile 对象应有的行为很相似（即要严格按实际情况去访问，不能随意优化 ）
* 所以，这些 workaround 函数就像是 volatile 本应提供的保护机制的一种 “冗余备份”，能在 volatile 编译出问题时，辅助保证对变量访问的正确性

## Takeaways

* volatile tells the compiler that accessing an object may have side effects that mustn’t be optimized away.
* The compiler must keep accesses to volatile objects in order, but may reorder accesses to non-volatile objects around them.
* Use synchronization tools (e.g., mutexes and semaphores) rather than volatile objects to manage inter-thread communication.
* Accesses to volatile objects are not guaranteed to be atomic.
