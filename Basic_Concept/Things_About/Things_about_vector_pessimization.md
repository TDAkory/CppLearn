# vector pessimization

> [What is the vector pessimization](https://quuxplusone.github.io/blog/2022/08/26/vector-pessimization/)

## vector尺寸调整

`std::vector<T>` 是一个动态分配的可调整大小的数组。每当 v 重新分配其 T 的缓冲区时，它总是分配其所需空间的两倍。实际上[godbolt](https://godbolt.org/z/Exc6W36Ke)，每个库供应商在实现上略有不同。GNU libstdc++ 使用上述算法；libc++ 基于旧的容量而不是旧的大小进行计算；而 Microsoft 的 STL 使用1.5的增长因子而不是2。

## 强异常保证

`std::vector`是C++98 STL的一部分，C++98时期还没有移动语义。因此，原始的C++98 `vector`中的“转移”步骤总是使用`T`的拷贝构造函数完成。每个旧缓冲区中的`T`都被复制到新缓冲区中；然后，指针被重新指向，旧的`T`被销毁，旧缓冲区被释放。

这种操作有一个很好的属性，即提供了**强异常保证**。在分配内存、任何拷贝操作等步骤中都可能抛出异常。但如果任何步骤抛出异常，我们可以简单地回滚：销毁已经构造的副本，释放新缓冲区，保持`v`的数据指针指向旧缓冲区。

这里的保证就是说：如果`push_back`抛出异常，那么所有数据仍然存在；就好像`push_back`操作从未发生过一样。

## 移动语义改变了局面

C++11引入了移动构造函数，作为拷贝构造函数的优化。移动构造函数`T(T&&)`接受一个右值引用，指向一个`T`，其有用生命周期即将结束，并可以从该`T`中“移动”数据，而不是昂贵地制作它们的独立副本。

在`std::vector`重新分配期间，向量旧缓冲区中的`T`本质上处于它们的有用生命周期的末尾。如果拷贝成功，我们只会销毁它们。因此，你可能会认为我们可以从这些`T`中“移动”数据，如下图所示：

![img](https://quuxplusone.github.io/blog/images/2022-08-26-realloc-via-move.png)

这里，绿色方框表示包含用户数据的`T`对象，红色方框表示已经被“移动”其内容并处于“移动后状态”的`T`对象。显然，这种方式在复制用户数据时浪费的时间更少！

但是，使用移动而不是拷贝会导致我们失去强异常保证。假设在图中所示的点抛出异常（由`T`的移动构造函数抛出）——在移动第二个或第三个元素期间。我们希望撤销操作并回滚到用户原来的向量，但我们不能：我们已经从第一个两个元素中移动了数据！如果此时销毁并释放新缓冲区，用户将只能得到他们原始数据的一部分。

### 4. 可能抛出异常的移动构造函数
你可能会说：“为什么C++11不简单地规定移动构造函数绝对不能抛出异常？我们可以将非抛出性作为移动构造函数的含义的一部分。毕竟，编写移动构造函数是可选的。如果你不能编写一个非抛出的移动构造函数，就根本不写一个！”但这也不行。

考虑`std::list<int>`。它是一个堆分配的链表，其中`lst.begin()`给你一个迭代器，指向列表的第一个堆分配的节点，而`lst.end()`给你一个指向列表的“最后一个加一”节点的一过末尾迭代器。对于“最后一个加一”节点存储在哪里，有两种实际的方法。libstdc++和libc++在`std::list`对象本身的内存占用内存储一个“虚拟节点”。但Microsoft的STL（基于Dinkumware的）在堆上存储一个额外的“哨兵节点”，与其他列表节点一起。因此，在MSVC上，`std::list`的默认构造函数和移动构造函数都需要分配一个哨兵节点，而这个内存分配可能会（假设性地）抛出异常。

“那么？如果`std::list`不能被无异常地移动，也许它根本就应该是不可移动的。”好吧，假设Microsoft的`std::list`没有定义任何移动构造函数。你仍然可以写`auto lst2 = std::move(lst);`——它最终会调用拷贝构造函数。

（见“‘通用引用’还是‘转发引用’？”（2022-02-02）。）这就是最初将移动构造函数视为拷贝构造函数的另一个重载的原因。

此外，我们确实希望`std::list`有一个移动构造函数。移动构造一个包含100个元素的`std::list`可能需要在堆上分配一个哨兵节点，但拷贝构造这个100个元素的`std::list`将需要在堆上分配101个节点！即使我们有时必须使它们可能抛出异常，移动语义仍然是一个巨大的胜利。

“我们能不能简单地强迫Microsoft重写他们的`std::list`实现，以匹配libstdc++和libc++？”理论上，可以。实际上，不行，一个重要的供应商不会仅仅因为你礼貌地请求就破坏他们的ABI。（还要记住，这是2003-2009年左右的事情，不是2022年。）

好吧，所以，C++11确实需要承认移动构造函数可能抛出异常的可能性。因此，`vector`的重新分配操作需要一种方法来区分抛出和非抛出的移动构造函数。基本上，`vector`需要做的是：

```cpp
if constexpr (is_nothrow_move_constructible_v<T>) {
    // 在循环中移动
} else {
    // 在循环中拷贝，并在异常时回滚
}
```

实现`is_nothrow_move_constructible`是一个问题。我们如何用C++解决一个问题？我们添加一个关键字。

### 5. `noexcept`
C++11采用的解决方案是引入了一个新关键字：`noexcept`。未标记`noexcept`的函数（即大多数函数）可以抛出异常。标记为`noexcept`的函数承诺它们永远不会抛出异常。实际上，这不仅仅是一个承诺：它也是一个防火墙。来自较低级别的异常永远不会传播到`noexcept`之外；它们会撞到它并用`std::terminate`终止程序。

现在，`vector`的重新分配操作可以简单地询问`T(T&&)`是否是`noexcept`。如果是，那么向量重新分配将使用移动构造并高效运行。如果不是，那么向量重新分配将使用拷贝构造，就像C++98一样慢——但如果真的抛出了异常，则保留强异常保证。

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

### 结论：向量性能退化
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

如果你不小心遗漏了`noexcept`，那么你将得到向量性能退化：每次重新分配`std::vector<Instrument>`时，它都会昂贵地进行拷贝，而不是廉价地进行移动。

更糟糕的是，有时——特别是在MSVC上，使用Microsoft的STL实现