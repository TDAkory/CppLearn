# [Assembler Instructions with C Expression Operands](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)

* [asm volatile内嵌汇编用法](https://blog.csdn.net/whut_gyx/article/details/39078339)

带有C/C++表达式的内联汇编格式为：

```cpp
　　__asm__　__volatile__("InSTructiON List" : Output : Input : Clobber/Modify);
```

* `_asm_` 是GCC关键字`asm`的宏定义：`#define __asm__  asm`。`__asm__`  或`asm`用来声明一个内联汇编表达式，任何内联汇编表达式都是以它开头，必不可少。
* Instruction list是汇编指令序列，可以为空比如：`__asm__ __volatile__("")`; 或 `__asm__ ("")`，都是完全正当的内联汇编表达式，只不过这两条语句没有什么意义。但是如：`__asm__ ("":::"memory")`，就有意义，它向GCC 声明：“内存作了改动”，GCC 在编译的时候，会将此因素考虑进去。当Instruction list中有多条指令时，可以将多条指令放在一对引号中，用`;`或`\n`将它们分开，如过一条指令放一对引号中，可以每条指令一行。即：（1）每条指令都必须被双引号括起来 (2)两条指令必须用换行或分号分开。如：
  
    ```cpp
        asm volatile("crc32 %%ebx, %%eax\n\t": "=a" (initval) : "b" (data), "a" (initval));
    ```

* `__volatile__`是GCC 关键字`volatile` 的宏定义：`#define __volatile__    volatile`。`__volatile__`或`volatile` 是可选的， 假如用了它，则是向GCC 声明不答应对该内联汇编优化，否则当 使用了优化选项(-O)进行编译时，GCC 将会根据自己的判定决定是否将这个内联汇编表达式中的指令优化掉。
* Output 用来指定内联汇编语句的输出
* Input 域的内容用来指定当前内联汇编语句的输入，Input中，格式为形如“constraint”(variable)的列表（逗号分隔)
* 有时候，你想通知GCC当前内联汇编语句可能会对某些寄存器或内存进行修改，希看GCC在编译时能够将这一点考虑进往。那么你就可以在Clobber/Modify域声明这些寄存器或内存。这种情况一般发生在一个寄存器出现在"Instruction List"，但却不是由Input/Output操纵表达式所指定的，也不是在一些Input/Output操纵表达式使用"r"约束时由GCC 为其选择的，同时此寄存器被"Instruction List"中的指令修改，而这个寄存器只是供当前内联汇编临时使用的情况。

