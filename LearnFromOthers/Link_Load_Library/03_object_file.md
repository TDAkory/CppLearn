# Object File

- [Object File](#object-file)
  - [ELF文件类型](#elf文件类型)
  - [目标文件](#目标文件)
    - [例子分析](#例子分析)
  - [ELF文件结构](#elf文件结构)
    - [文件头](#文件头)
    - [段表](#段表)
    - [重定位表](#重定位表)
    - [字符串表](#字符串表)
  - [链接的接口----符号](#链接的接口----符号)
    - [符号表](#符号表)


可执行文件的格式，主要是Windows下的PE（Portable Executable）和Linux的ELF（Executable Linkable Format），两者都是COFF（Common file format）的变种。

目标文件就是源代码编译后但未进行链接的那些中间文件 (Windows 的.obj 和Linux 下的.o)，它跟可执行文件的内容与结构很相似，所以一般跟可执行文件格式 一 起采用 种格式存储。从广义上看，目标文件与可执行文件的格式其实几乎是一样的，所以我们可以广义地将目标文件与可执行文件看成是一种类型的文件。

简言之，可执行文件、动态链接库（.dll .so）、静态链接库(.lib .a)都是按照可执行文件的格式存储的。

## [ELF文件类型](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#:~:text=In%20computing%2C%20the%20Executable%20and,shared%20libraries%2C%20and%20core%20dumps.)

ELF文件标准把系统中采取ELF格式的文件分为四类：

> 使用`file`命令查看文件格式

* 可重定位文件：包含代码和数据，可被用来链接成 Executable or Linkable，静态链接库也算这一类 （.o .obj）
* 可执行文件
* 共享目标文件（Shared Object File） （.so .dll）
* 核心转储文件（Core Dump File）

## 目标文件

* 一般C语言的编译后执行语句都编译成机器代码，保存在`.text`段:
* 已初始化的全局变量和局部静态变量都保存在`.data`段;
* 末初始化的全局变量和局部静态变量一般放在一个叫`.bss`段里。只是为未初始化的全局变量和局部静态变量预留位置而已，它并没有内容，所以它在文件中 也不占据空间。

程序指令 & 程序数据 分开的好处：
* 只读 & 读写权限不同，可以映射到不同的内存区域
* 现代CPU的指令缓存和数据缓存通常十分开的
* 共享的指令 数据，在多程序的环境可以节约大量内存

### 例子分析

> Linux n37-018-026 5.4.143.bsk.8-amd64 #5.4.143.bsk.8 SMP Debian 5.4.143.bsk.8 Wed Jul 20 08:43:36 UTC  x86_64 GNU/Linux

```c
int printf(const char *format, ...);

int global_init_var = 84;
int global_uninit_var;

void func1(int i){
	printf("%d\n", i);
}

int main(void) {
	static int static_var = 85;
	static int static_var2;

	int a = 1;
	int b;

	func1(static_var + static_var2 + a + b);

	return a;
}
```

```shell
$ gcc -c simple_section.c

$ objdump -h simple_section.o

simple_section.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000057  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000008  0000000000000000  0000000000000000  00000098  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  0000000000000000  0000000000000000  000000a0  2**2
                  ALLOC
  3 .rodata       00000004  0000000000000000  0000000000000000  000000a0  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000001d  0000000000000000  0000000000000000  000000a4  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  000000c1  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000058  0000000000000000  0000000000000000  000000c8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```

`.rodata` 只读数据段、`.comment` 注释信息段、`.note.GNU-stack` 堆栈提示段

`CONTENTS` 表示该段在文件中存在

如下命令可以查看各段长度，`dec`是长度的和的十进制，`hex`是长度的和的十六进制

```shell
$ size simple_section.o
   text	   data	    bss	    dec	    hex	filename
    179	      8	      4	    191	     bf	simple_section.o
```

data段保存初始化的全局静态变量和局部静态变量 `global_init_var` 和 `static_var`，刚好8字节，`"%d\n"`是只读的字符串，保存在rodata段。

bss段存放未初始化的全局变量和局部静态变量 `global_uninit_var` 和 `static_var2`，本质上是预留了空间。但实际只有4字节，通过符号表可以看到只有 `static_var2` 在这里，`global_uninit_var` 只是一个未定义的 `COMMON` 符号（与编译器实现有关）

`objdump` `-s` 将所有段的内容用十六进制打印，`-d`将包含指令的段反汇编

```shell
$ objdump -s -d simple_section.o

simple_section.o:     file format elf64-x86-64

Contents of section .text:
 0000 554889e5 4883ec10 897dfc8b 45fc89c6  UH..H....}..E...
 0010 488d3d00 000000b8 00000000 e8000000  H.=.............
 0020 0090c9c3 554889e5 4883ec10 c745fc01  ....UH..H....E..
 0030 0000008b 15000000 008b0500 00000001  ................
 0040 c28b45fc 01c28b45 f801d089 c7e80000  ..E....E........
 0050 00008b45 fcc9c3                      ...E...
Contents of section .data:
 0000 54000000 55000000                    T...U...
Contents of section .rodata:
 0000 25640a00                             %d..
Contents of section .comment:
 ...

Disassembly of section .text:

0000000000000000 <func1>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	8b 45 fc             	mov    -0x4(%rbp),%eax
   e:	89 c6                	mov    %eax,%esi
  10:	48 8d 3d 00 00 00 00 	lea    0x0(%rip),%rdi        # 17 <func1+0x17>
  17:	b8 00 00 00 00       	mov    $0x0,%eax
  1c:	e8 00 00 00 00       	callq  21 <func1+0x21>
  21:	90                   	nop
  22:	c9                   	leaveq
  23:	c3                   	retq

0000000000000024 <main>:
  24:	55                   	push   %rbp
  25:	48 89 e5             	mov    %rsp,%rbp
  28:	48 83 ec 10          	sub    $0x10,%rsp
  2c:	c7 45 fc 01 00 00 00 	movl   $0x1,-0x4(%rbp)
  33:	8b 15 00 00 00 00    	mov    0x0(%rip),%edx        # 39 <main+0x15>
  39:	8b 05 00 00 00 00    	mov    0x0(%rip),%eax        # 3f <main+0x1b>
  3f:	01 c2                	add    %eax,%edx
  41:	8b 45 fc             	mov    -0x4(%rbp),%eax
  44:	01 c2                	add    %eax,%edx
  46:	8b 45 f8             	mov    -0x8(%rbp),%eax
  49:	01 d0                	add    %edx,%eax
  4b:	89 c7                	mov    %eax,%edi
  4d:	e8 00 00 00 00       	callq  52 <main+0x2e>
  52:	8b 45 fc             	mov    -0x4(%rbp),%eax
  55:	c9                   	leaveq
  56:	c3                   	retq
```

> 使用 `objcopy` 可以将二进制文件，如图片、音频、词典等作为目标文件的一个段
>
> __attribute__((section("XXX"))) 可以指定目标对象存放的段位置

## ELF文件结构

| ELF Header |
| :--------: |
|   .text    |
|   .data    |
|   .bss     |
|   ...      |
|   other sections     |
| section header table |
| string tables |
| symbol tables |
|  ... |

### 文件头

```shell
$ readelf -h simple_section.o
ELF Header:                                                             // typedef struct {
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00              // unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)             //   Elf32_Half	e_type;			/* Object file type */
  Machine:                           Advanced Micro Devices X86-64      //   Elf32_Half	e_machine;		/* Architecture */
  Version:                           0x1                                //   Elf32_Word	e_version;		/* Object file version */
  Entry point address:               0x0                                //   Elf32_Addr	e_entry;		/* Entry point virtual address */
  Start of program headers:          0 (bytes into file)                //   Elf32_Off	e_phoff;		/* Program header table file offset */
  Start of section headers:          1096 (bytes into file)             //   Elf32_Off	e_shoff;		/* Section header table file offset */
  Flags:                             0x0                                //   Elf32_Word	e_flags;		/* Processor-specific flags */
  Size of this header:               64 (bytes)                         //   Elf32_Half	e_ehsize;		/* ELF header size in bytes */
  Size of program headers:           0 (bytes)                          //   Elf32_Half	e_phentsize;		/* Program header table entry size */
  Number of program headers:         0                                  //   Elf32_Half	e_phnum;		/* Program header table entry count */
  Size of section headers:           64 (bytes)                         //   Elf32_Half	e_shentsize;		/* Section header table entry size */
  Number of section headers:         13                                 //   Elf32_Half	e_shnum;		/* Section header table entry count */
  Section header string table index: 12                                 //   Elf32_Half	e_shstrndx;		/* Section header string table index */
                                                                        // } Elf32_Ehdr;
```

ELF文件头结构和相关常数被定义在`/usr/include/elf.h`，针对32位和64位的位宽差别，分别定义了 `Elf32_Ehdr` 和 `Elf64_Ehdr`。

### 段表

段表：除文件头外最重要的结构，描述各个段的信息：段名、长度、偏移、读写权限等，编译器、链接器和装载器都靠段表来定位。对应结构是 `Elf32_Shdr`

```shell
$ readelf -S -W simple_section.o
There are 13 section headers, starting at offset 0x448:

Section Headers:
  // name type virtual_addr_at_execution offset size_in_bytes entry_size link info align
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000000000 000040 000057 00  AX  0   0  1
  [ 2] .rela.text        RELA            0000000000000000 000338 000078 18   I 10   1  8
  [ 3] .data             PROGBITS        0000000000000000 000098 000008 00  WA  0   0  4
  [ 4] .bss              NOBITS          0000000000000000 0000a0 000004 00  WA  0   0  4
  [ 5] .rodata           PROGBITS        0000000000000000 0000a0 000004 00   A  0   0  1
  [ 6] .comment          PROGBITS        0000000000000000 0000a4 00001d 01  MS  0   0  1
  [ 7] .note.GNU-stack   PROGBITS        0000000000000000 0000c1 000000 00      0   0  1
  [ 8] .eh_frame         PROGBITS        0000000000000000 0000c8 000058 00   A  0   0  8
  [ 9] .rela.eh_frame    RELA            0000000000000000 0003b0 000030 18   I 10   8  8
  [10] .symtab           SYMTAB          0000000000000000 000120 000198 18     11  11  8
  [11] .strtab           STRTAB          0000000000000000 0002b8 00007d 00      0   0  1
  [12] .shstrtab         STRTAB          0000000000000000 0003e0 000061 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```

### 重定位表

`.rela.text` 其类型是 `RELA` ，即重定位表。链接器在处理目标文件时，需要对目标文件中的某些元素进行重定位，即代码段和数据段汇总那些对绝对地址的引用的位置，这些重定位的信息都几类在重定位表里。

对每个需要重定位的代码段或数据段，都会有一个重定位表。

例子中包含`printf`的调用，因此需要对代码段重定位；不存在外部定义的变量，因此不存在 `.rela.data`。

重定位表也是`ELF`中的一个段，它的`link`表示符号表的下标，`info`表示作用于哪个段。上面`.text`段序号位`1`，因此`.rela.text`的`info`为`1`

### 字符串表

字符串表（String Table `.strtab`）保存普通字符串，段表字符串表（Section Header String Table `.shstrtab`）保存段表中用到的字符串，比如段名。

对照 ELF header，`e_shstrndx` `Section header string table index: 12`，就是该段表在所有段表的下标，即 `[12] .shstrtab`。

因此通过 ELF header 和 段表，可以解析整个ELF文件。

## 链接的接口----符号

链接的本质是把多个不同的目标文件互相“粘合”在一起，因此需要一个固定的规则。**在链接中，目标文件互相拼合本质上是目标文件之间对函数和变量的地址的互相引用。函数和变量统称为符号。**

每个目标文件都有一个符号表，里面记录了目标文件中用到的所有符号，每个定义的符号都有一个对应的值，符号值，对变量和函数来说，符号值就是它们的地址。除此之外，还存在其他类型的符号：

* 定义在本目标文件的全局符号，可以被其他目标文件引用
* 在本目标文件中引用的全局符号，却没有定义在本目标文件，称之为外部符号
* 段名，通常由编译器产生
* 局部符号，static，仅在编译单元内部可见
* 行号信息，可选的，由编译器生成

使用`nm`检查例子的符号结果如下：

```shell
$ nm simple_section.o
0000000000000000 T func1
0000000000000000 D global_init_var
                 U _GLOBAL_OFFSET_TABLE_
0000000000000004 C global_uninit_var
0000000000000024 T main
                 U printf
0000000000000004 d static_var.1965
0000000000000000 b static_var2.1966
```

### 符号表

`.symtab` `Elf32_Sym`结构