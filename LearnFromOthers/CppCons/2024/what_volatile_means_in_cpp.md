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

