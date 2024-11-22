# 可执行文件的装载和进程

- [可执行文件的装载和进程](#可执行文件的装载和进程)
  - [进程虚拟地址空间](#进程虚拟地址空间)
  - [装载的方式](#装载的方式)
    - [覆盖装入](#覆盖装入)
    - [页映射](#页映射)
  - [从操作系统角度看可执行文件的装载](#从操作系统角度看可执行文件的装载)
  - [进程虚拟空间分布](#进程虚拟空间分布)
    - [Segment 和 Section](#segment-和-section)
    - [堆和栈](#堆和栈)
    - [堆的最大申请数量](#堆的最大申请数量)
    - [段地址对齐](#段地址对齐)
    - [进程栈初始化](#进程栈初始化)
  - [Linux内核装载ELF过程](#linux内核装载elf过程)


## 进程虚拟地址空间

每个程序被运行起来，都有自己独立的虚拟地址空间，其大小由计算机的硬件平台决定：CPU位数-->寻址空间的大小-->地址空间的理论上限。

以32位为例，虚拟地址空间大小为4GB，当程序并不能任意使用。默认的，高地址的1GB `0xC0000000 ~ 0xFFFFFFFF` 是操作系统分配给内核的，剩下的3GB，原则上是分配给进程使用的。

**PAE（Physical Address Extension）**：32位CPU下，程序能否使用超过4GB内存？如果指虚拟地址空间，不能，因为32位CPU只能使用32为指针，寻址范围被限定；如果指计算机的内存空间，可以，Intel自1995采用了36位的物理地址，可以访问64GB物理内存。得益于页映射方式的修改，可以访问到更多的物理内存，即PAE。

扩展的物理地址空间，应用程序是感知不到的。那么应当如何使用呢？一个常见的方式是操作系统提供一个窗口映射的方法，把这些额外的内存映射到进程地址空间中来，应用程序可以根据需要来选择申请和映射。比如`0x10000000~0x20000000`这一段256MB作为窗口，程序可以从高于4GB的物理地址空间中申请多个大小为256MB的块，按序映射到虚拟地址空间上来使用。Windows下，这种方式叫做AWE（Address Windowing Extensions），Linux下，通过mmap()实现。

## 装载的方式

程序执行时所需要的指令和数据必须在内存中才能正常运行，最简单的办法就是全部装入内存。

矛盾点在于内存可能不足。后来研究发现，可以利用局部性原理，将程序中最常用的部分驻留在内存中，将不太常用的数据房子啊磁盘上，按需加载，这就是动态装载的基本原理：覆盖装入（`Overlay`）、页映射（`Paging`）

### 覆盖装入

把挖掘内存迁离的任务交给程序员，在编写时将程序分隔成若干块，然后编写一个辅助代码来管理这些模块应该何时加载、何时被替换掉，被称为`Overlay Manager`。在复杂程序涉及多模块时，程序员必须手工将模块按照它们之间的调用依赖关系组织成树状结构。（要求： 每个调用路径上的模块必须同时在内存中、不允许跨树调用）。

执行效率和开发效率低，仅在某些受限场景下保留，大部分场景已经弃用。

### 页映射

> TODO 补充虚拟内存介绍

是虚拟存储机制的一部分。

将内存和所有磁盘中的数据和指令按照 页 为单位划分成若干个页，后续装载和操作的单位都是页。

## 从操作系统角度看可执行文件的装载

> 从操作系统视角，一个进程最关键的特征是拥有独立的虚拟地址空间

下面描述创建一个进程，然后装载相应的可执行文件并执行的过程

1. **创建一个独立的虚拟地址空间**：一个虚拟空间由一组页映射函数将虚拟空间的各个页映射到相应的物理空间，因此创建虚拟空间实质上并不是创建空间，二是创建映射函数所需要的相应数据结构。
2. **读取可执行文件头，建立虚拟地址空间与可执行文件的映射关系**；上一步的页映射关系函数是虚拟空间到物理内存的映射关系，这一步是虚拟空间与可执行文件的映射关系。
3. **将CPU的指令寄存器设置成可执行文件的入口地址，启动执行**：涉及内核堆栈和用户堆栈的切换、CPU运行权限的切换等

> 由于可执行文件在装载时实际上是被映射的虚拟空间，所以可执行文件很多时候又被叫做映像文件（Image）

这种映射关系只是保存在操作系统的一个数据结构，Linux中奖进程虚拟克难攻坚中的一个段叫做虚拟内存区域（VMA，Virtual Memory Area）。比如，操作系统创建进程后，会在进程相应的数据结构中设置一个`.text`段的`VMA`，它在虚拟空间中的地址为`0x08048000~0x08049000`，对应`ELF`文件中偏移为`0`的`.text`，它的属性是只读，etc

**页错误**：上面的步骤执行完以后，其实可执行文件的真正指令和数据都没有被装入到内存中。操作系统只是通过可执行文件头部的信息建立起可执行文件和进程虚存之间的映射关系而已。 假设在上面的例子中，程序的入又地址为`Ox08048000`，即刚好是`.text` 段的起始地址。当CPU开始打算执行这个地址的指令时，发现页面`0x08048000 ~0x08049000`是个空页面，于是它就认为这是一个页错误 (Page Fault)。CPU将控制权交给操作系统，操作系统有专门的页错误处理例程来处理这种情况。这时候我们前面提到的装载过程的第二步建立的数据结构起到了很关键的作用，操作系统将查询这个数据结构，然后找到空页面所在的VMA，计算出相应的页面在可执行文件中的偏移，然后在物理内存中分配一个物理页面，将进程中该虚拟页与分配的物理页之间建立映射关系，然后把控制权再还回给进程，进程从刚才页错误的位置重新开始执行。

随着进程的执行，页错误也会不断地产生，操作系统也会为进程分配相应的物理页面来 满足进程执行的需求。

> 操作系统虚拟内存管理 

## 进程虚拟空间分布

### Segment 和 Section

实际产生的可执行文件包含很多段，而操作系统映射时必须以页长度为单位，这就会带来内存的浪费。

实质上，操作系统并不关心可执行文件各个段所包含的实际内容，它只关心一些与装载相关的内容：最主要的是段的访问权限（可读、可写、可执行）。同时，在ELF文件中，权限仅有为数不多的几个组合：

* 以代码段为代表的可读可执行
* 以数据段和BSS段为代码的可读可写
* 以只读数据段代表的只读

因此，对于相同权限的段，可以把它们合并到一起当做一个段进行映射。`ELF`引入一个概念叫做`Segment`，一个`Segment`包含一个或多个属性类似的`Section`。描述`Section`属性的结构叫做段表，描述`Segment`的结构叫做程序头，它描述了`ELF`文件该如何被操作系统映射到进程的虚拟空间，通过`readelf -l xxx`可以查看：

```c
#include <stdlib.h>

int main() {
    while (1) {
        sleep(1000);
    }
    return 0;
}
```

gcc -static sleep.c  -o sleep.elf

```shell
> readelf -l sleep.elf -W

Elf file type is EXEC (Executable file)
Entry point 0x401a30
There are 8 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  LOAD           0x000000 0x0000000000400000 0x0000000000400000 0x000470 0x000470 R   0x1000
  LOAD           0x001000 0x0000000000401000 0x0000000000401000 0x07b081 0x07b081 R E 0x1000
  LOAD           0x07d000 0x000000000047d000 0x000000000047d000 0x0237ac 0x0237ac R   0x1000
  LOAD           0x0a10e0 0x00000000004a20e0 0x00000000004a20e0 0x0051f0 0x006940 RW  0x1000
  NOTE           0x000200 0x0000000000400200 0x0000000000400200 0x000044 0x000044 R   0x4
  TLS            0x0a10e0 0x00000000004a20e0 0x00000000004a20e0 0x000020 0x000060 R   0x8
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x0a10e0 0x00000000004a20e0 0x00000000004a20e0 0x002f20 0x002f20 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00     .note.ABI-tag .note.gnu.build-id .rela.plt 
   01     .init .plt .text __libc_freeres_fn .fini 
   02     .rodata .eh_frame .gcc_except_table 
   03     .tdata .init_array .fini_array .data.rel.ro .got .got.plt .data __libc_subfreeres __libc_IO_vtables __libc_atexit .bss __libc_freeres_ptrs 
   04     .note.ABI-tag .note.gnu.build-id 
   05     .tdata .tbss 
   06     
   07     .tdata .init_array .fini_array .data.rel.ro .got 
```

可以看到文件中有8个Segment，我们只需要关心 LOAD 类型的段，只有这些段是需要被映射的，其他的 Segment 都是在装载过程中起辅助作用的。

所以总的来说，`Segment`和`Section`是从不同的角度来划分同一个 `ELF` 文件。这个在 `ELF` 中被称为不同的视图(View)， 从“Section”的角度来看ELF文件就是链接视图(Linking View)， 从 “Segment” 的角度来看就是执行视图(Execution View )。 当我们在谈到ELF装载时，“段” 专门指“Segment”;而在其他的情况下，“段” 指的是“Section”。

ELF可执行文件中有一个数据结构叫做程序头表（Program Header Table），用来保存 Segment 信息

```c
/* Program segment header.  */

typedef struct
{
  Elf32_Word	p_type;			/* Segment type */
  Elf32_Off	    p_offset;		/* Segment file offset */
  Elf32_Addr	p_vaddr;		/* Segment virtual address */
  Elf32_Addr	p_paddr;		/* Segment physical address */
  Elf32_Word	p_filesz;		/* Segment size in file */
  Elf32_Word	p_memsz;		/* Segment size in memory */
  Elf32_Word	p_flags;		/* Segment flags */
  Elf32_Word	p_align;		/* Segment alignment */
} Elf32_Phdr;
```

### 堆和栈

堆和栈在进程的虚拟空间中的表现也是以VMA的形式存在的。大多数情况下，一个进程中的堆和栈分别有一个对应的VMA

```shell
> ./sleep.elf &
> ps -ef | grep sleep
zhaojie+ 3226777 3214925  0 17:03 ?        00:00:00 sleep 180
root     3227868     834  0 17:05 ?        00:00:00 sleep 60
zhaojie+ 3228130 3215970  0 17:05 pts/8    00:00:00 ./sleep.elf
zhaojie+ 3228234 3215970  0 17:05 pts/8    00:00:00 grep --color=auto sleep
> cat /proc/3228130/maps
00400000-00401000 r--p 00000000 fe:10 9182634                            /data00/home/zhaojieyi/MyTest/observe_link/sleep.elf
00401000-0047d000 r-xp 00001000 fe:10 9182634                            /data00/home/zhaojieyi/MyTest/observe_link/sleep.elf
0047d000-004a1000 r--p 0007d000 fe:10 9182634                            /data00/home/zhaojieyi/MyTest/observe_link/sleep.elf
004a2000-004a8000 rw-p 000a1000 fe:10 9182634                            /data00/home/zhaojieyi/MyTest/observe_link/sleep.elf
004a8000-004a9000 rw-p 00000000 00:00 0 
01816000-01839000 rw-p 00000000 00:00 0                                  [heap]
7ffe91011000-7ffe91032000 rw-p 00000000 00:00 0                          [stack]
7ffe91110000-7ffe91113000 r--p 00000000 00:00 0                          [vvar]
7ffe91113000-7ffe91114000 r-xp 00000000 00:00 0                          [vdso]
```

{VMA地址范围，VMA权限（r读w写x执行p私有s共享），偏移量，映像文件所在设备的主设备号和次设备号，映像文件节点，映像文件路径}

后面几个设备号和文件节点号为0的，叫做匿名虚拟内存区域，在这里可以看到堆和栈，以及两个内核空间的映射：vvar vdso

总的来说，操作系统通过给进程划分一组VMA来管理进程的虚拟空间：基本原则是相同权限属性的、有相同映像文件的映射成一个VMA，一个进程基本包含以下几种：

* 代码VMA
* 数据VMA
* 堆VMA
* 栈VMA

### 堆的最大申请数量

```c
#include <stdio.h>
#include <stdlib.h>

unsigned maximum = 0;

int main(int argc, char *argv[]) {
    unsigned blocksize[] = {1024 * 1024, 1024, 1};
    int i, count;
    for (i = 0; i < 3; i++) {
        for (count = 1;; count++) {
            void *block = malloc(maximum + blocksize[i] * count);
            if (block) {
                maximum = maximum + blocksize[i] * count;
                free(block);
            }
            else {
                break;
            }
        }
    }

    printf("%u bytes\n", maximum);
}
```

### 段地址对齐

 装载时通过虚拟内存的页映射机制完成的。在8086系统中，默认页的大小是4K字节。因此要将一段物理内存和进程的虚拟地址空间建立映射关系，要求：

1. 内存空间的长度是4096字节的整数倍
2. 物理内存和进程虚拟地址空间的起始地址必须是4096的整数倍

如何处理内存碎片？UNIX系统采用了一种连续映射的方式（即让各个段接壤的部分共享一个物理页面，然后将该物理页面分别映射两次）。而且UNIX系统将ELF的文件头页看做一个段，也会映射到进程的地址空间。这样做的好处是进程中的某一段区域就是整个ELF文件的映像。

### 进程栈初始化

参数压栈，esp指向栈顶

## Linux内核装载ELF过程

1. 用户层面，bash进程会调用fork()系统调用创建一个新进程
2. 新进程调用execve()系统调用执行指定的ELF文件
3. 原来的bash进程继续返回等待刚才启动的新进程结束，然后继续等待用户输入命令

```c
// /usr/include/unistd.h
/* Replace the current process, executing PATH with arguments ARGV and
   environment ENVP.  ARGV and ENVP are terminated by NULL pointers.  */
extern int execve (const char *__path, char *const __argv[],
		   char *const __envp[]) __THROW __nonnull ((1, 2));
```

```shell
execve()
    sys_execve()
        do_execve()
            查找被执行文件，找到后读取文件的前128字节，判断文件类型
            search_binary_handle() 查找可执行文件装载处理过程
            load_elf_binary()
                检查ELF文件格式的有效性
                查找动态链接段`.interp`，设置动态链接器路径
                根据ELF文件的程序头描述，对ELF文件进行映射
                初始化ELF进程环境（比如EDX寄存器的地址应该是DT_FINI的地址）
                将系统调用的返回地址修改成ELF可执行文件的入口点（静态链接的ELF可执行文件，入口点就是e_entry；动态链接的ELF可执行文件，入口点是动态链接器）
    EIP指向了ELF程序的入口地址，新程序开始执行