# `vector`性能退化和`noexcept`

> 译自[What is the vector pessimization](https://quuxplusone.github.io/blog/2022/08/26/vector-pessimization/)

## vector尺寸调整

`std::vector<T>` 是一个动态分配的可调整大小的数组。每当 `v` 重新分配其 `T` 的缓冲区时，它总是分配其所需空间的两倍。实际上[godbolt](https://godbolt.org/z/Exc6W36Ke)，每个库供应商在实现上略有不同。`GNU libstdc++` 使用上述算法；`libc++` 基于旧的容量而不是旧的大小进行计算；而 `Microsoft` 的 `STL` 使用1.5的增长因子而不是2。

## 强异常保证

`std::vector`是C++98 STL的一部分，C++98时期还没有移动语义。因此，原始的C++98 `vector`中的“转移”步骤总是使用`T`的拷贝构造函数完成。每个旧缓冲区中的`T`都被复制到新缓冲区中；然后，指针被重新指向，旧的`T`被销毁，旧缓冲区被释放。

这种操作有一个很好的属性，即提供了**强异常保证**。在分配内存、任何拷贝操作等步骤中都可能抛出异常。但如果任何步骤抛出异常，我们可以简单地回滚：销毁已经构造的副本，释放新缓冲区，保持`v`的数据指针指向旧缓冲区。

这里的保证就是说：如果`push_back`抛出异常，那么所有数据仍然存在；就好像`push_back`操作从未发生过一样。

## 移动语义改变了局面

C++11引入了移动构造函数，作为拷贝构造函数的优化。移动构造函数`T(T&&)`接受一个右值引用，指向一个`T`，入参的生命周期即将结束，可以从该`T`中“移动”数据，而不是拷贝一份。

但是，使用移动而不是拷贝会导致我们失去强异常保证。假设在移动的过程中（部分元素完成了移动，还有部分元素尚未移动）发生了异常，我们希望撤销操作并回滚到用户原来的向量。但这无法做到：我们已经从第一个两个元素中移动了数据！如果此时销毁并释放新缓冲区，用户将只能得到他们原始数据的一部分。

> 是否可以考虑利用`swap`将数据转移回原始的位置？理论上，`swap`不会`throw`，但实际上，`C++`并不把`swap`看作是原始运算。如果被`swap`的类型`T`的移动构造函数抛出异常，那么`swap<T>`也会抛出异常，这是因`为swap<T>`就是基于移动语义实现的。

需要注意，这里无法回滚的根源是`T(T&&)`会抛出异常。如果`vector`能够确定其特化的类型`T(T&&)`不抛出异常，那么也就不需要回滚，使用移动语义也是安全的。

## 可能抛出异常的移动构造函数

一个显然会想到的问题是：“为什么C++11不简单地规定移动构造函数绝对不能抛出异常？我们可以将非抛出性作为移动构造函数的含义的一部分。毕竟，编写移动构造函数是可选的。如果你不能编写一个非抛出的移动构造函数，就根本不写一个！”

但这也不行。一句话描述原因就是：标准委员会没办法让编译器供应商达成一致

比如`std::list<int>`。它是一个堆分配的链表，其中`lst.begin()`给你一个迭代器，指向列表的第一个堆分配的节点，而`lst.end()`给你一个指向列表的“最后一个加一”节点的一过末尾迭代器。对于“最后一个加一”节点存储在哪里，有两种实际的方法。`libstdc++`和`libc++`在`std::list`对象本身的内存占用内存储一个“虚拟节点”。但`Microsoft`的STL（基于Dinkumware的）在堆上存储一个额外的“哨兵节点”，与其他列表节点一起。因此，在`MSVC`上，`std::list`的默认构造函数和移动构造函数都需要分配一个哨兵节点，而这个内存分配可能会（假设性地）抛出异常。

“我们能不能简单地强迫`Microsoft`重写他们的`std::list`实现，以匹配libstdc++和libc++？”理论上，可以。实际上，不行，一个重要的供应商不会仅仅因为你礼貌地请求就破坏他们的ABI。

C++11确实需要承认移动构造函数可能抛出异常的可能性。因此，`vector`的重新分配操作需要一种方法来区分抛出和非抛出的移动构造函数。基本上，`vector`需要做的是：

```cpp
if constexpr (is_nothrow_move_constructible_v<T>) {
    // 在循环中移动
} else {
    // 在循环中拷贝，并在异常时回滚
}
```

如何实现`is_nothrow_move_constructible`？C++标准的做法是通过添加一个新的关键字。

## `noexcept`

`noexcept`：未标记`noexcept`的函数（即大多数函数）可以抛出异常。标记为`noexcept`的函数承诺它们永远不会抛出异常。实际上，这不仅仅是一个承诺：它也是一个防火墙。来自较低级别的异常永远不会传播到`noexcept`之外；它们会撞到它并用`std::terminate`终止程序。

现在，`vector`的重新分配操作可以简单地询问`T(T&&)`是否是`noexcept`。如果是，那么重新分配将使用移动构造并高效运行。如果不是，那么重新分配将使用拷贝构造，就像C++98一样慢，但如果真的抛出了异常，则保留强异常保证。

`noexcept`以自然的方式通过（隐式或显式）默认成员传播：

```cpp
struct A { A(A&&); };
struct B { B(B&&) noexcept; };
struct JustB { B b; JustB(JustB&&) = default; };
struct AB { A a; B b; };

static_assert(!std::is_nothrow_move_constructible_v<A>);
static_assert(std::is_nothrow_move_constructible_v<B>);
static_assert(std::is_nothrow_move_constructible_v<JustB>);
static_assert(!std::is_nothrow_move_constructible_v<AB>);
```

`AB`的隐式默认移动构造函数是可能抛出的，因为移动`a`成员可能会抛出异常。`JustB`的显式默认移动构造函数是`noexcept`的，因为它只需要移动`B`。

## 结论：向量性能退化

如果你为自己的类型编写了一个用户定义的移动构造函数，无论它做什么，如果它的主体不是`=default`，你必须标记它为`noexcept`。

```cpp
struct Instrument {
    int n_;
    std::string s_;

    Instrument(const Instrument&) = default;

    // 错误！
    Instrument(Instrument&& rhs)
        : n_(std::exchange(rhs.n_, 0)),
          s_(std::move(s_))
        {}

    // 正确！
    Instrument(Instrument&& rhs) noexcept
        : n_(std::exchange(rhs.n_, 0)),
          s_(std::move(s_))
        {}
};
```

如果你不小心遗漏了`noexcept`，那么将得到vector性能退化：每次重新分配`std::vector<Instrument>`时，会进行昂贵的拷贝，而不是廉价地移动。

那么在`MSVC`的编译环境下，就无法使用移动构造来进行vector的重新分配了么？

```cpp
struct Widget {
    std::list<int> data_;
    explicit Widget(int n) : data_(n) {}
};
```

上述示例在`MSVC`编译器下，`Widget`将获得一个非`noexcept`的默认移动构造函数，因此会遇到向量性能退化问题。为了修复这个问题，可以显式声明：

```cpp
struct Gadget {
    std::list<int> data_;
    explicit Gadget(int n) : data_(n) {}

    Gadget(Gadget&&) noexcept = default;
    Gadget(const Gadget&) = default;
    Gadget& operator=(Gadget&&) = default;
    Gadget& operator=(const Gadget&) = default;
};
```

这会告诉编译器，我希望`Gadget`的移动构造函数执行默认操作，同时它是`noexcept`的。因此可以避免向量性能退化。

注意，如果`std::list`的移动构造函数真的耗尽内存，并在`Gadget(Gadget&&)`内部抛出`std::bad_alloc`异常，那么这个异常将撞上我们人为设置的`noexcept`防火墙，并终止程序。这也是一种权衡，在性能退化和内存耗尽时退出之间。不过对大多数程序来说，`std::bad_alloc`并不是一个真正的担忧，而`vector::push_back`的速度可能是！

综上我们可以得到以下原则：

* 如果类型有一个用户定义的移动构造函数，应该总是将该移动构造函数声明为noexcept。
* 如果“Rule-of-Zero”类型有任何非无抛出移动的数据成员（在`MSVC`上尤其常见），应该考虑为其显式声明一个标记为noexcept的默认移动构造函数。
* 如果都不行，则应该了解`vector`性能退化问题：知道`std::vector<T>`在重新分配期间可以将移动操作转换为拷贝操作，当且仅当`T(T&&)`没有被标记为`noexcept`。

## Ref

* [Self-assignment and the Rule of Zero](https://quuxplusone.github.io/blog/2019/08/20/rule-of-zero-pitfall/)
* [Using noexcept](https://akrzemi1.wordpress.com/2011/06/10/using-noexcept/)
