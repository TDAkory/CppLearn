# 深入理解`std::shared_ptr`

`std::shared_ptr` 是 `C++11` 引入的智能指针，用于自动管理动态分配内存的生命周期，多个 `std::shared_ptr` 实例可以共享同一个对象，对象会在最后一个引用它的智能指针被销毁时自动被删除。

这样的一句话描述似乎非常好理解，不过深究之下，`std::shared_ptr`依然有不少值得思考的细节。

## 一些背景

智能指针的概念并非始于 `C++11`。早在 1990 年代，Boost 库就引入了各种智能指针实现，包括 `boost::shared_ptr`。这些早期实现为 C++ 标准委员会提供了宝贵的经验和设计参考。C++11 标准中的智能指针很大程度上借鉴了 `Boost` 库的设计，但在细节上进行了改进和优化。

`std::shared_ptr` 的核心价值在于它实现了**共享所有权**（shared ownership）的概念，具体包含以下几点：

1. **资源生命周期管理**：自动跟踪对象的引用情况，确保对象在不再需要时被释放。
2. **避免内存泄漏**：即使在复杂的控制流或异常处理场景下，也能确保资源的正确释放。
3. **简化接口设计**：明确表达对象所有权语义，使 API 设计更加清晰。
4. **支持复杂的资源管理场景**：例如对象之间的循环引用（通过配合 `std::weak_ptr` 使用）。

`C++11` 标准库提供了`std::shared_ptr`在内的三种智能指针，`C++14` 和 `C++17` 标准进一步增强了这些智能指针的功能，例如添加了 `std::make_unique`（`C++14`）和对数组类型的更好支持（`C++17`）,`C++20` 则引入了 `std::atomic<std::shared_ptr>`，进一步提升了多线程环境下的使用体验。

## 实现细节

从设计角度讲，`shared_ptr` 旨在平衡安全性、效率和灵活性这三个关键目标：

1. **安全性**：
   - 自动内存管理，避免内存泄漏和悬空指针
   - 线程安全的引用计数操作，确保在多线程环境中安全使用
   - 类型安全，避免类型转换错误

2. **效率**：
   - 最小化运行时开销，特别是在引用计数操作上
   - 优化内存布局，提高缓存局部性
   - 提供如 `make_shared` 等工具，减少内存分配次数

3. **灵活性**：
   - 支持自定义删除器，允许管理各种资源，而不仅仅是堆内存
   - 支持自定义分配器，允许控制内存分配策略
   - 提供类型转换和别名构造等高级功能，适应复杂的使用场景

我们参考 GNU 的 libstbc++ 实现来分析 `std::shared_ptr` 的相关细节。

### 继承关系

```cpp
// shared_ptr.h
template<typename _Tp>
class shared_ptr : public __shared_ptr<_Tp> {
    using element_type = typename __shared_ptr<_Tp>::element_type;
    friend class weak_ptr<_Tp>;
    ……
};

// shared_ptr_base.h
template<typename _Tp, _Lock_policy _Lp>
class __shared_ptr : public __shared_ptr_access<_Tp, _Lp> {
    using element_type = typename remove_extent<_Tp>::type;
    ……
    element_type*   _M_ptr;              // Contained pointer.
    __shared_count<_Lp>  _M_refcount;    // Reference counter.
};


// Define operator* and operator-> for shared_ptr<T>.
template<typename _Tp, _Lock_policy _Lp, bool = is_array<_Tp>::value, bool = is_void<_Tp>::value>
class __shared_ptr_access {
    using element_type = _Tp;
    ……
  private:
    element_type* _M_get() const noexcept { 
        return static_cast<const __shared_ptr<_Tp, _Lp>*>(this)->get(); 
    }
};

template <_Lock_policy _Lp> class __shared_count {
  public:
    template <typename _Ptr> explicit __shared_count(_Ptr __p) : _M_pi(0) {
        __try {
            _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);
        } __catch(...) {
            delete __p;
            __throw_exception_again;
        }
    }
    ……
    _Sp_counted_base<_Lp> *_M_pi;
};

// Counted ptr with no deleter or allocator support
template <typename _Ptr, _Lock_policy _Lp>
class _Sp_counted_ptr final : public _Sp_counted_base<_Lp> { 
    …… 
    _Ptr _M_ptr;
};

// Support for custom deleter and/or allocator
template <typename _Ptr, typename _Deleter, typename _Alloc, _Lock_policy _Lp>
class _Sp_counted_deleter final : public _Sp_counted_base<_Lp> { 
    class _Impl : _Sp_ebo_helper<0, _Deleter>, _Sp_ebo_helper<1, _Alloc> {
        _Ptr _M_ptr;
    };
    …… 
    _Impl _M_impl;
};

template <typename _Tp, typename _Alloc, _Lock_policy _Lp>
class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp> { …… };

template <_Lock_policy _Lp = __default_lock_policy>
class _Sp_counted_base : public _Mutex_base<_Lp> {
    ……
    _Atomic_word _M_use_count;  // #shared
    _Atomic_word _M_weak_count; // #weak + (#shared != 0)
};
```

![shared_ptr structure](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CppLearn/shared_ptr.jpg)

可以看到，`std::shared_ptr`是面向用户的一层封装，其实现核心在`__shared_ptr`中：

* `__shared_ptr`中的 `_M_ptr` 是指向被管理对象的指针，
* `__shared_ptr`中的 `_M_refcount` 则包含了指向控制块的指针以及相关的引用计数操作，其类型是`__shared_count`
  * `__shared_count`的构造入参也包含了一个`_Tp`类型的指针，被保存在其成员变量`_Sp_counted_base`中
  * `_Sp_counted_base`的类型是`_Sp_counted_base`，有多个派生，用来支持模版类型的默认析构函数或自定义析构函数
* `__shared_ptr`继承了 `__shared_ptr_access`，这是一个访问接口的封装

### 内存开销分析

在 64 位系统上，一个 `shared_ptr` 对象通常占用 16 字节的内存（两个 8 字节指针）。这比原始指针（8 字节）大一倍。值得注意的是，虽然 `shared_ptr` 本身的大小是固定的（两个指针的大小），但它所关联的控制块的大小可能会根据创建方式的不同而变化。例如，当使用自定义删除器或分配器时，控制块会更大，因为它需要存储这些额外的组件。简单的概括如下：

1. **`shared_ptr` 对象本身**：占用两个指针的大小（通常为 16 字节），而裸指针只占用一个指针的大小（通常为 8 字节）。
2. **控制块**：每组共享同一对象的 `shared_ptr` 实例会共享一个控制块，控制块的大小取决于其内容，但通常至少包含：
   - 强引用计数（通常为 4 或 8 字节）
   - 弱引用计数（通常为 4 或 8 字节）
   - 删除器（大小取决于删除器类型）
   - 可能的分配器（大小取决于分配器类型）
   - 可能的对象本身（当使用 `make_shared` 时）

### 控制块的设计和实现

## 一些值得思考的问题

### `shard_ptr`构造和`make_shared`的区别

### 控制块内的指针和别名构造（Aliasing Constructor）

### `enable_shared_from_this`

## 引用

* [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr.html)
* [shared_ptr.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr.h)
* [shared_ptr_atomic.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_atomic.h)
* [shared_ptr_base.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_base.h)