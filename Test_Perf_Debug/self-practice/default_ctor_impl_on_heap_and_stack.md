# default\_ctor\_impl\_on\_heap\_and\_stack

> 开发过程中需要使用默认移动构造函数，并涉及到堆栈内存的操作，故一探究竟

## 默认移动构造函数的条件

启用默认移动构造函数必须满足以下全部条件：

* 没有声明拷贝赋值函数。 
* 没有声明拷贝构造函数。 
* 没有声明移动赋值函数。
* 移动构造函数没有隐式声明为delete
* **没有声明析构函数。**

对于类内的成员对象，如果是STL容器这种本身有移动实现的成员变量，又不包含非RAII的资源管理，那么默认移动逻辑是可以预想到的，但是对于堆栈内存，移动构造会进行何种处理，还不是很明白，因此写代码进行验证。

## 代码验证

```cpp
#include <iostream>

using namespace std;

class BufferA {
public:
    BufferA(){}

    // 显式指明使用默认的移动构造函数
    BufferA(BufferA &&b) = default;    
    BufferA& operator=(BufferA &&b) = default;

    char stackbuf_[256];
    char *heapbuf_;
};

void TestBufferA() {
    static const string stackStr = "It is on Stack.";
    static const string heapStr = "It is on Heap.";
    BufferA bufA;
    bufA.heapbuf_ = static_cast<char *>(malloc(64));
    memcpy(bufA.heapbuf_, heapStr.c_str(), heapStr.length());
    memcpy(bufA.stackbuf_, stackStr.c_str(), stackStr.length());

    cout << "stackbuf: " << static_cast<void *>(bufA.stackbuf_) << endl;
    cout << "heapbuf: " << static_cast<void *>(bufA.heapbuf_) << endl;
    cout << "stackStr: " << bufA.stackbuf_ << endl;
    cout << "heapStr: " << bufA.heapbuf_ << endl;

    BufferA bufB(move(bufA));
    cout << "After move" << endl;
    cout << "bufB" << endl;
    cout << "stackbuf: " << static_cast<void *>(bufB.stackbuf_) << endl;
    cout << "heapbuf: " << static_cast<void *>(bufB.heapbuf_) << endl;
    cout << "stackStr: " << bufB.stackbuf_ << endl;
    cout << "heapStr: " << bufB.heapbuf_ << endl;
    cout << "bufA" << endl;
    cout << "stackbuf: " << static_cast<void *>(bufA.stackbuf_) << endl;
    cout << "heapbuf: " << static_cast<void *>(bufA.heapbuf_) << endl;
    cout << "stackStr: " << bufA.stackbuf_ << endl;
    cout << "heapStr: " << bufA.heapbuf_ << endl;
}

int main()
{
    TestBufferA();
}
```

## 运行结果

```text
stackbuf: 0x7ffeeabdd970
heapbuf: 0x7fa4ec405800
stackStr: It is on Stack.
heapStr: It is on Heap.
After move
bufB
stackbuf: 0x7ffeeabdd868
heapbuf: 0x7fa4ec405800
stackStr: It is on Stack.
heapStr: It is on Heap.
bufA
stackbuf: 0x7ffeeabdd970
heapbuf: 0x7fa4ec405800
stackStr: It is on Stack.
heapStr: It is on Heap.
```

## 结论

1、对于栈内存，因为是跟随变量的，因此默认移动构造的效果是memcpy

2、对于堆内存，因为类成员仅是一个指针，因此默认移动构造的效果是指针赋值，内存空间重复使用

3、将 BufferA\(BufferA &&b\) = default; 和 BufferA& operator=\(BufferA &&b\) = default; 注释掉运行，也可以得到同样的结果，证明在满足条件的情况下，是否显式指定默认移动构造函数对编译器而言没有区别

