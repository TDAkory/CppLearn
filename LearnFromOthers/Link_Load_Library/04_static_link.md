# 静态链接

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
