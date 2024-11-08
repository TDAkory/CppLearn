# 静态链接

- [静态链接](#静态链接)
  - [例子](#例子)
  - [空间和地址分配](#空间和地址分配)
  - [符号解析和重定位](#符号解析和重定位)
    - [重定位](#重定位)
    - [指令修正方式](#指令修正方式)
  - [COMMON块](#common块)
  - [C++相关问题](#c相关问题)
    - [重复代码消除](#重复代码消除)
    - [函数级别链接](#函数级别链接)
  - [Ref](#ref)


## 例子

```c
// a.c
extern int shared;

int main() {
    int a = 100;
    swap(&a, &shared);
}

// b.c
int shared = 1;

void swap(int *a, int *b) {
    *a ^= *b ^= *a ^= *b;
}
```

## 空间和地址分配

链接的过程，就是将几个输入目标文件加工后合并成一个输出文件。显而易见的问题是：多个输入目标文件，如何将其各个段合并到输出文件？

**按序叠加**：一种简单的办法就是将输入的各个目标文件按照次序叠加起来（碎片化、浪费空间）

**相似段合并**：将相同性质的段合并到一起

“链接器为目标文件分配地址和空间” 这句话中的“地址和空间” 其实有两个含义 :

* 第一个是在输出的可执行文件中的空间
* 第二个是在装载后的虚拟地址中的虚拟地址空间。
  
对于有实际数据的段，比如`.text`和`.data`来说，它们在文件中和虚拟地址中都要分配空间，因为它们在这两者中都存在；而对于`.bss`这样的段来说，分配空间的意义只局限于虚拟地址空间，因为它在文件中并没有内容。事实上，**我们在这里谈到的空间分配只关注于虚拟地址空间的分配**，因为这个关系到链接器后面的关于地址计算的步骤，而可执行文件本身的空间分配与链接过程关系并不是很大。

相似段合并通常需要两步：

1. 空间和地址分配
2. 符号解析和重定位

```shell
$ gcc -c a.c b.c
a.c: In function ‘main’:
a.c:5:5: warning: implicit declaration of function ‘swap’ [-Wimplicit-function-declaration]
     swap(&a, &shared);
     ^~~~

$ ld a.o b.o -e main -o ab

$ ls
ab  a.c  a.o  b.c  b.o

$ objdump -h a.o

a.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000002e  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000006e  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000006e  2**0
                  ALLOC
  3 .comment      0000001d  0000000000000000  0000000000000000  0000006e  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  0000008b  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  00000090  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

$ objdump -h b.o

b.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000004b  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  0000000000000000  0000000000000000  0000008c  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000090  2**0
                  ALLOC
  3 .comment      0000001d  0000000000000000  0000000000000000  00000090  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000ad  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  000000b0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA

$ objdump -h ab

ab:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .note.gnu.property 00000020  0000000000400190  0000000000400190  00000190  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text         00000079  0000000000401000  0000000000401000  00001000  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .eh_frame     00000058  0000000000402000  0000000000402000  00002000  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .data         00000004  0000000000404000  0000000000404000  00003000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  4 .comment      0000001c  0000000000000000  0000000000000000  00003004  2**0
                  CONTENTS, READONLY
```

链接前后的程序中所使用的地址已经是程序在进程中的虚拟地址了（VMA、 Size）。在链接前，目标文件的VMA都是0，因为虚拟地址空间还没有分配。链接后，`.text`段被分配在地址 `0x0000000000401000`， 长度是`0x79`，`.data`段被分配在地址`0x0000000000404000`，长度是`0x4`。

符号的虚拟地址，则通过 `输出文件段地址 + 输入文件段在输出文件段的偏移 + 符号在段内的偏移` 进行计算

## 符号解析和重定位

### 重定位

a.c中使用了shared和swap，那么编译器在编译a.c的时候，是如何处理这两个符号的呢？

```shell
$ objdump -d a.o

a.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 83 ec 10             sub    $0x10,%rsp
   8:   c7 45 fc 64 00 00 00    movl   $0x64,-0x4(%rbp)
   f:   48 8d 45 fc             lea    -0x4(%rbp),%rax
  13:   48 8d 35 00 00 00 00    lea    0x0(%rip),%rsi        # 1a <main+0x1a>
  1a:   48 89 c7                mov    %rax,%rdi
  1d:   b8 00 00 00 00          mov    $0x0,%eax
  22:   e8 00 00 00 00          callq  27 <main+0x27>
  27:   b8 00 00 00 00          mov    $0x0,%eax
  2c:   c9                      leaveq 
  2d:   c3                      retq 
```

这里在未进行空间分配之前，main的起始地址是0x0，同时，shared和swap两个符号的地址，也是0x0，可以从0x13、0x22两条指令看到

```shell
$ objdump -d ab

ab:     file format elf64-x86-64


Disassembly of section .text:

0000000000401000 <main>:
  401000:       55                      push   %rbp
  401001:       48 89 e5                mov    %rsp,%rbp
  401004:       48 83 ec 10             sub    $0x10,%rsp
  401008:       c7 45 fc 64 00 00 00    movl   $0x64,-0x4(%rbp)
  40100f:       48 8d 45 fc             lea    -0x4(%rbp),%rax
  401013:       48 8d 35 e6 2f 00 00    lea    0x2fe6(%rip),%rsi        # 404000 <shared>
  40101a:       48 89 c7                mov    %rax,%rdi
  40101d:       b8 00 00 00 00          mov    $0x0,%eax
  401022:       e8 07 00 00 00          callq  40102e <swap>
  401027:       b8 00 00 00 00          mov    $0x0,%eax
  40102c:       c9                      leaveq 
  40102d:       c3                      retq   

000000000040102e <swap>:
  ......
```

对比可以看到，`shared`的地址是`0x00002fe6`，`swap`的地址是`0x00000007`。这里如何理解呢？`callq`指令其实是一条近地址相对位移调用指令（Call near, relative, displacement relative to next instruction）,也就是说，`callq`后面的值表示跳转地址相对于下一条指令的偏移量。在上面例子中，`callq`的下一条指令是`mov`，地址是`0x401027`，则`swap`的地址是 `0x401027 + 0x7 = 0x40102e`。

**重定位表**：在ELF文件中，保存与重定位相关的信息，是一个或多个段。每一个包含需要重定位符号的段，都会生成对应的重定位段。

```shell
$ readelf -r a.o

Relocation section '.rela.text' at offset 0x218 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000016  000900000002 R_X86_64_PC32     0000000000000000 shared - 4
000000000023  000b00000004 R_X86_64_PLT32    0000000000000000 swap - 4

Relocation section '.rela.eh_frame' at offset 0x248 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```

每个要被重定位的地方被称之为一个重定位入口（Relocation Entry），其偏移表示该入口在要被重定位的段中的位置。从反汇编可以看到，这里的`0x16` 和 `0x23` 就是前面 `lea`指令 和 `callq`指令的操作数地址。

重定位表的结构：

```c
typedef struct
{
  Elf32_Addr	r_offset;		/* Address */
  Elf32_Word	r_info;			/* Relocation type and symbol index */
  Elf32_Sword	r_addend;		/* Addend */
} Elf32_Rela;
```

`r_info`的低8位表示重定位入口的类型，高位表示重定位入口的符号在符号表中的下标。以`shared`符号为例，它的类型是`#define R_X86_64_PC32		2	/* PC relative 32 bit signed */`，这是一个32位的相对地址，同时从下面目标文件`a.o`的符号表可以看到，`shared`的序号是9，与info展示的信息是对应的

**符号解析**的过程则是在链接时，针对每一个重定位入口，去全局符号表中查找目标文件地址的过程。比如如下符号表，`shared`和`swap`都是`UND`未定义的，需要在链接器扫描完所有的输入目标文件之后，这些未定义符号都能够在全局符号表中找到，否则会报符号未定义错误。

```shell
$ readelf -s a.o

Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS a.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000000    46 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND shared
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND swap

$ readelf -s ab

Symbol table '.symtab' contains 14 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000400190     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000401000     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000402000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000404000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS b.c
     7: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS a.c
     8: 000000000040102e    75 FUNC    GLOBAL DEFAULT    2 swap
     9: 0000000000404000     4 OBJECT  GLOBAL DEFAULT    4 shared
    10: 0000000000404004     0 NOTYPE  GLOBAL DEFAULT    4 __bss_start
    11: 0000000000401000    46 FUNC    GLOBAL DEFAULT    2 main
    12: 0000000000404004     0 NOTYPE  GLOBAL DEFAULT    4 _edata
    13: 0000000000404008     0 NOTYPE  GLOBAL DEFAULT    4 _end
```

### 指令修正方式

对重定位符号地址的计算方式有很多种，大体上存在以下的区别：

* 近址寻址 远址寻址
* 绝对寻址 相对寻址
* 寻址长度是8、16、32、64位

绝对寻址修正后的地址就是该符号的实际地址，相对寻址修正后的地址为符号距离被修正位置的地址差

还是以shared为例，其重定位类型为`R_X86_64_PC32`表示重定位符号引用的结果是一个 32 位的 PC 相对地址。

一个 PC 相对地址就是距程序计数器（即PC寄存器）的当前运行时值的偏移量。当 CPU 执行一条使用 32 位 PC 相对寻址的指令时，它会将指令中编码的 32 位值加上 PC 寄存器的当前运行值，作为目的地址（如call指令的目标地址），PC 寄存器中的值通常是下一条指令在内存中的地址。

因此，可以得出公式1：`ADDR(PC) + VALUE(reference) = ADDR(defined_symbol)`。

注：`ADDR(PC)`表示符号引用所在指令的下一条指令的运行时地址；`VALUE(reference)`表示要修改的符号引用的值，重定位符号引用的目的就是计算出该值的大小；`ADDR(defined_symbol)`表示被引用的符号定义的运行时地址。

另外，公式2：`ADDR(PC) + VALUE(r.addend) = ADDR(s) + VALUE(r.offset) = ADDR(reference)`。（该公式的最左侧部分是根据下文中 `VALUE(reference)` 的计算方式反向推导而来的，即最左侧部分为 `ADDR(PC) + VALUE(r.addend)`，而不是 `ADDR(PC) - VALUE(r.addend)`

注：`VALUE(r.addend)`表示重定位条目中`addend`字段的值；`ADDR(s)`表示符号引用的目标 `section` 的运行时地址（即符号引用所在 `section` 中第一个字节的虚拟地址）；`VALUE(r.offset)`表示重定位条目中offset字段的值。

根据公式 1 和公式 2，可以得出`VALUE(reference)`的计算方式。推导过程为：

```shell
VALUE(reference) = ADDR(defined_symbol) - ADDR(PC)
                 = ADDR(defined_symbol) - ( ADDR(s) + VALUE(r.offset) - VALUE(r.addend) )
                 = ADDR(defined_symbol) + VALUE(r.addend) - ( ADDR(s) + VALUE(r.offset) )
                 = ADDR(defined_symbol) + VALUE(r.addend) - ADDR(reference) 
```

注：`ADDR(reference)`表示需要修改的符号引用的运行时地址。

对于`shared`而言，其`VALUE(r.offset)=0x16`, `VALUE(r.addend)=-0x04`，它所在的段main得起始地址 `ADDR(s)=0x401000`，`ADDR(defined_symbol)=0x404000`

因此 `VALUE(reference) = 0x404000 - 0x04 - (0x401000 + 0x16) = 0x2fe6`

**简单表述相对寻址的计算为：`S+A-P`(S是符号地址，A是addend值，P是符号引用所在的地址，即被修正的位置)**

对比反汇编输出 `401013:       48 8d 35 e6 2f 00 00    lea    0x2fe6(%rip),%rsi        # 404000 <shared>`

小端序的地址 `e6 2f 00 00`，是完全一致的。

## COMMON块

需要 `COMMON` 机制的原因是编译器和链接器允许不同类型的弱符号存在，但最本质的原因还是 **链接器不支持符号类型**，即链接器无法判断各个符号的类型是否一致。

因此在编译节点，编译器无法确定一个弱符号的具体大小（有可能在别处定义了同名的强符号、或定义了占用更大空间的弱符号），因此编译器无法为未初始化的全局变量这类弱符号在BSS段分配空间。

当链接器读取所有输入的目标文件后，任何一个弱符号的最终大小都是可以确定的了，所以最终会在输出文件的BSS段分配空间。

GCC的`-fno-common`.也允许我们把所有来初始化的全局变量不以`COMMON`块的形 式处理，或者使用`int global __attribute__((nocommon));`
一旦一个未初始化的全局变量不是以`COMMON`块的形式存在，那么它就相当于一个强符号，如果其他目标文件中还有同一个变量的强符号定义，链接时就会发生符号重复定义错误。

## C++相关问题

### 重复代码消除

模板实例化、外部内联函数、虚函数表都有可能在不同的编译单元里生成相同的代码。最简单的情况就拿模板来说，模板从本质上讲很像宏，当模板在一个编译单元里被实例化时 ，它并不知道自己是否在别的编译单元也被实例化了。所以当一个模板在多个编译单元同时实例化成相同的类型的时候，必然会生成重复的代码。

当然，最简单的方案就是不管这些，将这些重复的代码都保留下来，但会带来：

* 空间浪费
* 地址容易出错
* 指令运行效率低，cache命中率低

实践中通常采用的做法是将每个模板的实例代码都单独防在一个段里，每个段只包含一个模板实例。比如有个模板函数是`add<T>()`，某个编译单元以`int`实例化该模板，就生成一个`.temp.add<int>`的段，这样在链接过程中，即便其他的编译单元也用`int`实例化了这个模板，链接器也能识别到，将它们合并进代码段。

### 函数级别链接

## Ref

- [计算机系统篇之链接（5）：静态链接（下）——重定位](https://csstormq.github.io/blog/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%B3%BB%E7%BB%9F%E7%AF%87%E4%B9%8B%E9%93%BE%E6%8E%A5%EF%BC%885%EF%BC%89%EF%BC%9A%E9%87%8D%E5%AE%9A%E4%BD%8D)