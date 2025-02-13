# [位域](https://learn.microsoft.com/zh-cn/cpp/cpp/cpp-bit-fields?view=msvc-170)

- [C/C++ 位域知识小结](https://www.cnblogs.com/zlcxbb/p/6803059.html)

类和结构可包含比整型类型占用更少存储空间的成员。 这些成员被指定为位域。 位域成员声明符规范的语法如下：

```cpp
declarator:constant-expression
```

**宽度为 0 的未命名位域强制将下一个位域与下一个类型边界对齐，其中类型是成员的类型。**

```cpp
// bit_fields1.cpp
// compile with: /LD
struct Date {
   unsigned short nWeekDay  : 3;    // 0..7   (3 bits)
   unsigned short nMonthDay : 6;    // 0..31  (6 bits)
   unsigned short nMonth    : 5;    // 0..12  (5 bits)
   unsigned short nYear     : 8;    // 0..100 (8 bits)
};
```

![BitField Above](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/bitfield.png)

nYear 长度为 8 位，这会溢出声明类型 unsigned short的单词边界。 因此，它从新 unsigned short的开头开始。 不需要所有位字段都适合基础类型的一个对象;根据声明中请求的位数分配新的存储单位。

1. 一个位域必须存储在同一个字节中，不能跨两个字节，故位域的长度不能大于一个字节的长度。
2. 取地址操作符&不能应用在位域字段上;
3. 位域字段不能是类的静态成员;
4. 位域字段在内存中的位置是按照从低位向高位的顺序放置的;
5. 位域的对齐

   1. 如果相邻位域字段的类型相同，且其位宽之和小于类型的sizeof大小，则后面的字段将紧邻前一个字段存储，直到不能容纳为止；

   2. 如果相邻位域字段的类型相同，但其位宽之和大于类型的sizeof大小，则后面的字段将从新的存储单元开始，其偏移量为其类型大小的整数倍；

   3. 如果相邻的两个位域字段的类型不同,则各个编译器的具体实现有差异,VC6采取不压缩方式,GCC和Dev-C++都采用压缩方式;

   4. 整个结构体的总大小为最宽基本类型成员大小的整数倍。

   5. 如果位域字段之间穿插着非位域字段，则不进行压缩；（不针对所有的编译器）

6. 当要把某个成员说明成位域时,其类型只能是int,unsigned int与signed int三者之一(说明:int类型通常代表特定机器中整数的自然长度。short类型通常为16位,long类型通常为32位,int类型可以为16位或32位.各编译器可以根据硬件特性自主选择合适的类型长度.见The C Programming Language中文 P32)。
