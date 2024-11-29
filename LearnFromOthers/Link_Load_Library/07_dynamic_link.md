# 动态链接

静态链接的问题是空间浪费、卡法和发布会受到依赖库的影响。

动态链接则不对组成程序的目标文件进行链接，等到程序要运行时才进行链接。好处：

* 减少空间浪费
* 依赖库升级方便，不需要整体重新编译
* 程序在运行时可以动态地加载各种程序模块（插件）
* 加强程序兼容性，在程序和运行平台之间的中间层

## 例子

```c
// program1.c
#include "Lib.h"

int main() {
    foobar(1);
    return 0;
}

// program2.c 
#include "Lib.h"

int main()
{
    foobar(2);
    return 0;
}

// Lib.c
#include <stdio.h>

void foobar(int i)
{
    printf("Printing from Lib.so %d\n", i);
    sleep(-1);
}

// Lib.h
#ifndef LIB_H
#define LIB_H

void foobar(int i);

#endif
```

```shell
$ gcc -fPIC --shared -o Lib.so Lib.c
$ gcc -o program1 program1.c ./Lib.so 
$ gcc -o program2 program2.c ./Lib.so
```

通过观察运行仅在的虚拟内存分布可以看到，系统把：`program1` `Lib.so` `libc.so` `ld-2.28.so` 都映射到了进程的地址空间。因此对于一个动态链接的可执行文件，系统在开始`program1`之前，先把控制权交给动态链接器，由它完成所有链接工作之后，再把控制权交给`program1`，执行程序。

查看Lib.so的装载属性：

```shell
$ readelf -l -W Lib.so

Elf file type is DYN (Shared object file)
Entry point 0x1060
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x0004d0 0x0004d0 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000151 0x000151 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x0000bc 0x0000bc R   0x1000
  LOAD           0x002e10 0x0000000000003e10 0x0000000000003e10 0x000220 0x000228 RW  0x1000
  DYNAMIC        0x002e20 0x0000000000003e20 0x0000000000003e20 0x0001c0 0x0001c0 RW  0x8
  NOTE           0x000238 0x0000000000000238 0x0000000000000238 0x000024 0x000024 R   0x4
  GNU_EH_FRAME   0x00201c 0x000000000000201c 0x000000000000201c 0x000024 0x000024 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002e10 0x0000000000003e10 0x0000000000003e10 0x0001f0 0x0001f0 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt 
   01     .init .plt .plt.got .text .fini 
   02     .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .dynamic .got 
```

和普通文件的不同点是：文件类型不同、起始地址是0x00000000，是一个无效地址。

实际装载地址并不是0x00000000，因此，**共享对象的最终装载地址在编译时是不确定的**

## 地址无关代码

**共享对象在编译时不能假设自己在进程虚拟地址空间中的位置**。可执行文件基本可以确定自己在进程虚拟空间中的起始地址，因为它通常是第一个被加载的，可以选择一个固定空闲的地址。

### 装载时重定位

装载时明确起始地址，利用偏移计算符号的实际装载地址。相对应的是前面提到的链接时重定位

缺陷是：指令无法再多个进程之间共享，因为每个进程装载时，空闲的地址可能是不同的，装载这个共享库时查找到的空闲的起始地址可能也是不同的，这对于指令部分在多个进程之间共享是不利的。

当然，动态链接库中的可修改数据部分对于不同进程来说有多个副本，所以可以采用装载时重定位来处理。

仅使用 `-shared` 就是装载时重定位

### 地址无关代码技术

希望程序模块中共享的指令部分在装载时不需要因为装载地址的改变而改变，所以实现的基本想法就是把指令中那些需要被修改的部分分离出来，跟数据部分防在一起，这样指令部分就可以保持不变，数据部分可以在每个进程中拥有一个副本。这个方案就是地址无关代码（PIC，Position-independent Code）技术。

一般模块中的地址引用方式，可以分为以下四种：

**1. 模块内的函数调用、跳转**

被调用的函数和调用者在同一个模块，之间的相对位置是固定的。可以用相对地址调用，或者是基于寄存器的相对调用，这种指令是不需要重定位的。

**2. 模块内部的数据访问，如模块中定义的全局变量、静态变量**

指令中不能包含数据的绝对地址，唯一的办法就是相对寻址。任何一条指令与它需要访问的模块内部数据之间的相对位置是固定的，只需要相对于当前指令加上固定的偏移量就可以访问模块内部数据了。

**3. 模块外部的函数调用、跳转**

与下面类型的方式一样，GOT中保存的是目标函数的地址。这种方法简单，但存在一些性能问题，实际上ELF采用了更复杂和精巧的方法，参见后面对动态链接优化的分析。

**4. 模块外部的数据访问，如其他模块定义的全局变量**

其他模块的全局变量的地址是跟模块装载地址有关的。ELF的做法是在**数据段里面建立一个指向这些变量的指针数组**，也被称为**全局偏移表**（Global Offset Table，GOT）当代码需要引用该全局变量时，可以通过GOT中相对于的项间接引用。

GOT本质上就是模块内部的数据，利用它做了一次中转。

### 如何确认一个动态库是否是PIC的

`readelf -d xxx.so | grep TEXTREL` 没有输出

因为PIC不应该包含代码段重定位表，TEXTREL表示代码段重定位表地址。

### 全局变量问题

```c
// module.c
extern int global;
int foo() {
    global = 1;
}
```

编译器在编译上面文件时，无法根据这个上下文判断`global`是定义在同一个模块的其他目标文件还是定义在另外一个共享对象中，即无法判断是否为跨模块间的调用。

假设module.c 是程序可执行文件的一部分，那么在这种情况下，由于程序主模块的代 码并不是地址无关代码，也就是说代码不会使用这种类似于PIC的机制，它引用这个全局变量的方式跟普通数据访问方式一样，编译器会产生这样的代码:
`movl $0x1,XXXXXXXX`
`XXXXXXXX`就是 `global` 的地址。由于可执行文件在运行时并不进行代码重定位，所以变量的地址必须在链接过程中确定下来。为能够使得链接过程正常进行，链接器会在创建可执行文件时，在它的`.bss`段创建一个 `global` 变量的副本。那么问题就很明显了，现在 `global` 变量定义在原先的共享对象中，而在可执行文件的`.bss`段还有一个副本。如果同一个变量同时存在于多个位置中，这在程序实际运行过程中肯定是不可行的。

于是解决的办法只有一个，那就是所有的使用这个变量的指令都指向位于可执行文件中的那个副本。`ELF`共享库在编译时，默认都把定义在模块内部的全局变量当作定义在其他模块的全局变量，也就是说当作前面的类型四，通过 `GOT` 来实现变量的访问。当共享模块被装载时，如果某个全局变量在可执行文件中拥有副本，那么动态链接器就会把 `GOT`中的相应地址指向该副本，这样该变量在运行时实际上最终就只有一个实例。如果变量在共享模块中被初始化，那么动态链接器还需要将该初始化值复制到程序主模块中的变量副本；如果该全局变量在程序主模块中没有副本，那么`GOT` 中的相应地址就指向模块内部的该变量副本。

假设module.c是一个共享对象的一部分，那么GCC编译器在`-fPIC`的情况下，就会把 对`global` 的调用按照跨模块模式产生代码。原因也很简单:编译器无法确定对`global` 的引用 是跨模块的还是模块内部的。即使是模块肉部的，即模块内部的全局变量的引用，按照上面 的结论，还是会产生跨模块代码，因为`global` 可能被可执行文件引用，从而使得共享模块中 对`global` 的引用要执行可执行文件中的`global` 副本。

### 数据段地址无关性

装载时重定位的共享对象的运行速度要比使用地址无关代码的共享对象快，因为它省去了地址无关代码中每次访问全局数据和函数时需要做一次计算当前地址以及间接地址寻址的过程。

## 延迟绑定

动态链接会略微慢一些，因为：对于全局和静态的数据访问都要进行复杂的GOT定位然后间接寻址；对于模块间的调用也要先定位GOT再进行间接跳转；程序启动需要进行运行时链接工作，启动会略慢。

**延迟绑定（Lazy Binding）**：当函数第一次被用到时才进行绑定（符号查找、重定位等），如果没有用到则不进行绑定。

ELF使用PLT(Procedure Linkage Table)来实现延迟绑定。

在Glibc中，完成地址绑定工作的函数是`_dl_runtime_resolve()`.

当我们调用某个外部模块的函数时，如果按照通常的做法应该是通过 GOT 中相应的项 进行间接跳转。PLT 为了实现延迟绑定，在这个过程中间又增加了一层间接跳转。调用函数通过一个叫做 PLT 项的结构进行跳转。每个外部函数在 PLT 中都有一个相应的项，比如：

```shell
bar@plt:
jmp *(bar@GOT)
push n
push moduleID
jump _dl_runtime_resolve
```

bar@plt 的第一条指令是一条通过GOT间接跳转的指令。bar@GOT 表示 GOT 中保存 bar()这个函数相应的项。如果链接器在初始化阶段已经初始化该项，并且将 bar() 的地址填入该项，那么这个跳转指令的结果就是我们所期望的，跳转到 bar()，实现函数正确调用。 但是为了实现延迟绑定，链接器在初始化阶段并没有将bar()的地址填入到该项，而是将上面代码中第二条指令`push n`的地址填入到`bar@GOT`中，这个步骤不需要查找任何符号，所以代价很低。很明显，第一条指令的效果是跳转到第二条指令，相当于没有进行任何操作。第二条指令将一个数字n压入堆栈中，这个数字是 bar 这个符号引用在重定位表 `rel.plt`中的下标。接着又是一条push指令将模块的ID压入到堆栈，然后跳转到`_dl_runtime _resolve`。这实际上就是在实现我们前面提到的`lookup(module, function)`这个函数的调用：先将所需要决议符号的下标压入堆栈，再将模块 ID 压入堆栈，然后调用动态链接器的 `_dl_runtime_resolve()` 来完成符号解析和重定位工作。

一旦`bar()`这个函数被解析完毕，当我们再次调用`bar@plt` 时，第一条jmp指令就能够跳转到真正的 `bar()`函数中，`bar()`函数返回的时候会根据堆栈里面保存的 EIP 直接返回到调用者，而不会再继续执行`bar@plt` 中第二条指令开始的那段代码，那段代码只会在符号未被解析时执行一次。

上述其实是理想化的模型，真实情况会略复杂一些。

`ELF` 把 `GOT` 拆分成了两个表 `.got` 和 `.got.plt` ，其中`.got`用来保存全局变量引用的地址，`.got.plt`用来保存函数引用的地址。此外，`.got.plt`的前三项还有特殊含义，分别如下：

1. `.dynamic`段的地址
2. 保存本模块的ID
3. 保存 `_dl_runtime_resolve()`的地址

第二项和第三项有动态加载器在装载共享模块时负责初始化。PLT结果也与理想化模型稍有不同，为了减少代码重复，ELF把最后两条指令防在了PLE的第一项，并规定每一项的长度是16字节，刚好存放3条指令：

```shell
PLT0:
push *(GOT + 4)
jump *(GOT + 8)
...
bar@plt
jump *(bar@GOT)
push n
jump PLT0
```

`PLT`在`ELF`中以单独的段存放，段名通常叫做 `.plt` ，因为其本身是一些地址无关的代码，所以可以与代码段合并成一个可读可执行的 `Segment` 被装载到内存。

## 动态链接相关结构

### `.interp`段

在ELF文件中决定该文件需要的动态链接器的路径，保存在`.interp`段。操作系统在对可执行文件进行加载的时候，会去寻找装载该可执行文件所需要的动态链接器，即该段指定路径的共享对象。

```shell
> objdump -s program1

program1:     file format elf64-x86-64

Contents of section .interp:
 02a8 2f6c6962 36342f6c 642d6c69 6e75782d  /lib64/ld-linux-
 02b8 7838362d 36342e73 6f2e3200           x86-64.so.2.    
```

也可以通过如下命令检查一个文件需要的动态链接器的路径

```shell
> readelf -l program1 | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

### `.dynamic`段

该段中的数据，对应结构 `Elf32_Dyn`，可以看作是动态链接下 ELF 文件的文件头，保存诸如：依赖哪些共享对象、动态链接符号表的位置 等信息

```c
/* Dynamic section entry.  */

typedef struct
{
  Elf32_Sword	d_tag;			/* Dynamic entry type */
  union
    {
      Elf32_Word d_val;			/* Integer value */
      Elf32_Addr d_ptr;			/* Address value */
    } d_un;
} Elf32_Dyn;
```

可以查看该段的内容

```shell
> readelf -d program1

Dynamic section at offset 0x2de8 contains 27 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [./Lib.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x1000
 0x000000000000000d (FINI)               0x11b4
 0x0000000000000019 (INIT_ARRAY)         0x3dd8
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x3de0
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x308
 0x0000000000000005 (STRTAB)             0x3d8
 0x0000000000000006 (SYMTAB)             0x330
 0x000000000000000a (STRSZ)              141 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x4000
 0x0000000000000002 (PLTRELSZ)           24 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x558
 0x0000000000000007 (RELA)               0x498
 0x0000000000000008 (RELASZ)             192 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffb (FLAGS_1)            Flags: PIE
 0x000000006ffffffe (VERNEED)            0x478
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x466
 0x000000006ffffff9 (RELACOUNT)          3
 0x0000000000000000 (NULL)               0x0
```

### 动态符号表`dynsym`

保存了动态链接的模块之间的导入导出的符号。此外还有动态符号字符串表`dynstr`，以及为了加速运行时符号查找的符号哈希表`.hash`。

### 动态链接重定位表

共享对象需要重定位的主要原因是导入符号的存在。

在编译时，这些符号的地址是未知的。在静态链接中，链接器最终链接时修正这些地址引用。但是在动态链接中，导入符号的地址是在运行时才确定的，所以需要再运行时将这些导入符号的引用修正，也就是重定位。

对于使用PIC技术的可执行文件或共享对象来说，虽然它们的代码段不需要重定位(因为地址无关)，但是数据段还包含了绝对地址的引用，因为代码段中绝对地址相关的部分被分离了出来，变成了GOT，而GOT实际上是数据段的一部分。除了GOT以外，数据段还可能包含绝对地址引用。

* 静态链接：`.rel.text`代码段重定位表 `.rel.data`数据段重定位表
* 动态链接：
  * `.rel.dyn`对数据引用的修正，所修正的位置在`.got`以及数据段
  * `.rel.plt`对函数引用的修正，所修正的位置在`.got.plt`


```shell
> readelf -r Lib.so 

Relocation section '.rela.dyn' at offset 0x3f8 contains 7 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003e10  000000000008 R_X86_64_RELATIVE                    1110
000000003e18  000000000008 R_X86_64_RELATIVE                    10d0
000000004028  000000000008 R_X86_64_RELATIVE                    4028
000000003fe0  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000003fe8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x4a0 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000004018  000200000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
000000004020  000500000007 R_X86_64_JUMP_SLO 0000000000000000 sleep@GLIBC_2.2.5 + 0
```

函数的重定位入又是不是只会出现在`.rel.plt`，而不会出现在`.rel.dyn`呢? 答案为否。如果某个ELF 文件是以PIC模式编译的(动态链接的可执行文件一般是PIC的)， 并调用了外部函数`bar`，则`bar`会出现在`.rel.plt` 中；而如果不是以PIC模式编详， 则bar将出现在`.rel.dyn` 中。

### 动态链接时进程堆栈初始化信息

动态链接器需要知道：可执行文件有几个Segment、每个Segment的属性、程序的入口地址等，这些信息有操作系统传递，保存在进程的堆栈中。

被称为辅助信息数组（Auxiliary Vector）

```c
/* This vector is normally only used by the program interpreter.  The
   usual definition in an ABI supplement uses the name auxv_t.  The
   vector is not usually defined in a standard <elf.h> file, but it
   can't hurt.  We rename it to avoid conflicts.  The sizes of these
   types are an arrangement between the exec server and the program
   interpreter, so we don't fully specify them here.  */

typedef struct
{
  uint32_t a_type;		/* Entry type */
  union
    {
      uint32_t a_val;		/* Integer value */
      /* We use to have pointer elements added here.  We cannot do that,
	 though, since it does not work when using 32-bit definitions
	 on 64-bit platforms and vice versa.  */
    } a_un;
} Elf32_auxv_t;
```

## 动态链接的步骤和实现

启动动态链接器本身、装载所有需要的共享对象、重定位和初始化

### 动态链接器自举

