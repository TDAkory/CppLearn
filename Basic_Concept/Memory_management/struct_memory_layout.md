# Struct Memory Layout

1. 自然对齐：如果一个变量的内存地址正好位于它长度的整数倍，就被称做自然对齐.如果不自然对齐，会带来CPU存取数据时的性能损失。
2. struct 的起始地址需要能够被其成员中最宽的基本数据类型整除；
3. struct 的 size 也必须能够被其成员中最宽的基本数据类型整除；
4. struct 中每个成员地址相对于struct 的起始地址的offset，必须是自然对齐的

```cpp
struct MyStruct
{
    char dda;   //偏移量为0，满足对齐方式，dda占用1个字节；
    double dda1;//下一个可用的地址的偏移量为1，不是sizeof(double)=8的倍数，需要补足7个字节才能使偏移量变为8（满足对齐方式），
                //因此VC自动填充7个字节，dda1存放在偏移量为8
                //的地址上，它占用8个字节。
    int type;   //下一个可用的地址的偏移量为16，是sizeof(int)=4的倍数，满足int的对齐方式，
                //所以不需要VC自动填充，type存
                //放在偏移量为16的地址上，它占用4个字节。
};
//所有成员变量都分配了空间，空间总的大小为1+7+8+4=20，不是结构的节边界数（即结构中占用最大空间的类型所占用的字节数sizeof(double)=8）的倍数，
// 所以需要填充4个字节，以满足结构的大小为sizeof(double)=8的倍数。
```