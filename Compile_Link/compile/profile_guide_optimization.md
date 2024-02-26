# [Profile-guided optimization](https://en.wikipedia.org/wiki/Profile-guided_optimization)

* 是一个通用的优化策略，主要有赖于各编译器的支持

- [Profile Guided Optimization](https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization)


编译反馈优化通常包括以下手段：

* Inlining，例如函数 A 频繁调用函数 B，B 函数相对小，则编译器会根据计算得出的 threshold 和 cost 选择是否将函数 B inline 到函数 A 中。
* ICP（Indirect call promotion），如果间接调用（Call Register）非常频繁地调用同一个被调用函数，则编译器会插入针对目标地址的比较和跳转指令。使得该被调用函数后续有了 inlining 和更多被优化机会，同时增加了 icache 的命中率，减少了分支预测的失败率。
* Register allocation，编译器能使用运行时数据做更好的寄存器分配。
* Basic block optimization，编译器能根据基本块的执行次数进行优化，将频繁执行的基本块放置在接近的位置，从而优化 data locality，减少访存开销。
* Size/speed optimization，编译器根据函数的运行时信息，对频繁执行的函数选择性能高于代码密度的优化策略。
* Function layout，类似于 Basic block optimization，编译器根据 Caller/Callee 的信息，将更容易在一条执行路径上的函数放在相同的段中。
* Condition branch optimization，编译器根据跳转信息，将更容易执行的分支放在比较指令之后，增加icache 命中率。
* Memory intrinsics，编译器根据 intrinsics 的调用频率选择是否将其展开，也能根据 intrinsics 接收的参数优化 memcpy 等 intrinsics 的实现。
