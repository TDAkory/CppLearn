# 深入理解`std::shared_ptr`

- [深入理解`std::shared_ptr`](#深入理解stdshared_ptr)
  - [一些背景](#一些背景)
  - [实现细节](#实现细节)
    - [继承关系](#继承关系)
    - [内存开销分析](#内存开销分析)
    - [控制块的设计和实现](#控制块的设计和实现)
      - [控制块的类型](#控制块的类型)
      - [创建规则](#创建规则)
      - [内存管理](#内存管理)
      - [线程安全](#线程安全)
      - [线程安全的边界](#线程安全的边界)
      - [内存序的应用](#内存序的应用)
      - [线程安全性相关的实现细节](#线程安全性相关的实现细节)
    - [C++20 中的 `std::atomic<std::shared_ptr>`](#c20-中的-stdatomicstdshared_ptr)
    - [`make_shared` 的实现原理](#make_shared-的实现原理)
  - [一些值得思考的问题](#一些值得思考的问题)
    - [`shard_ptr`构造和`make_shared`的区别](#shard_ptr构造和make_shared的区别)
    - [控制块内的指针和别名构造（Aliasing Constructor）](#控制块内的指针和别名构造aliasing-constructor)
    - [`enable_shared_from_this`](#enable_shared_from_this)
  - [最佳实践](#最佳实践)
  - [引用](#引用)


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

我们参考 GNU 的 libstbc++ 实现来分析 `std::shared_ptr` 的相关细节，相关源码：

- [shared_ptr_base.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_base.h)
- [shared_ptr_atomic.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_atomic.h)
- [shared_ptr.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr.h)

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

![shared_ptr structure](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/CppLearn/shared_ptr.png)

可以看到，`std::shared_ptr`是面向用户的一层封装，其实现核心在`__shared_ptr`中：

* `__shared_ptr`中的 `_M_ptr` 是指向被管理对象的指针，
* `__shared_ptr`中的 `_M_refcount` 则包含了指向控制块的指针以及相关的引用计数操作，其类型是`__shared_count`
  * `__shared_count`的构造入参也包含了一个`_Ptr`类型的指针，被保存在其成员变量`_Sp_counted_base`中，这里的类型实际是构造参数的类型
  * `_Sp_counted_base`的类型是`_Sp_counted_base`，有多个派生，用来支持模版类型的默认析构函数或自定义析构函数
* `__shared_ptr`继承了 `__shared_ptr_access`，这是一个访问接口的封装

### 内存开销分析

在 64 位系统上，一个 `shared_ptr` 对象通常占用 16 字节的内存（两个 8 字节指针）。这比原始指针（8 字节）大一倍。值得注意的是，虽然 `shared_ptr` 本身的大小是固定的（两个指针的大小），但它所关联的控制块的大小可能会根据创建方式的不同而变化。例如，当使用自定义删除器或分配器时，控制块会更大，因为它需要存储这些额外的组件。简单的概括如下：

1. **`shared_ptr` 对象本身**：占用两个指针的大小（通常为 16 字节），而裸指针只占用一个指针的大小（通常为 8 字节）。
2. **控制块**：每组共享同一对象的 `shared_ptr` 实例会共享一个控制块，控制块的大小取决于其内容，在64位系统上，不考虑自定义删除器的简单场景下，每次创建一个 `shared_ptr` 都会为控制块 `new` 一块 24 字节的内存。控制块可能包含以下内容：
   - 强引用计数（通常为 4 或 8 字节）
   - 弱引用计数（通常为 4 或 8 字节）
   - 删除器（大小取决于删除器类型）
   - 可能的分配器（大小取决于分配器类型）
   - 可能的对象本身（当使用 `make_shared` 时）

### 控制块的设计和实现

#### 控制块的类型

控制块的基类是 `_Sp_counted_base` ，这个基类定义了引用计数的基本操作，包括增加和减少引用计数、获取引用计数值等。

```cpp
template <_Lock_policy _Lp = __default_lock_policy>
class _Sp_counted_base : public _Mutex_base<_Lp> {
{
public:
    _Sp_counted_base() noexcept
        : _M_use_count(1), _M_weak_count(1) { }
    
    // 其他成员函数...

private:
    _Atomic_word  _M_use_count;     // #shared
    _Atomic_word  _M_weak_count;    // #weak + (#shared != 0)
};
```

从这个基类派生出几种不同类型的控制块实现，用于处理不同的初始化场景。

1. **`_Sp_counted_ptr`**：最简单的控制块类型，用于管理通过 `new` 表达式创建的对象，不包含自定义删除器或分配器。

    ```cpp
    template<typename _Ptr, _Lock_policy _Lp>
    class _Sp_counted_ptr final : public _Sp_counted_base<_Lp>
    {
    public:
        explicit _Sp_counted_ptr(_Ptr __p) noexcept
            : _M_ptr(__p) { }
        
        virtual void _M_dispose() noexcept
        { delete _M_ptr; }
        
        // 其他成员函数...
        
    private:
        _Ptr _M_ptr;  // 指向被管理对象的指针
    };
    ```

2. **`_Sp_counted_deleter`**：用于管理带有自定义删除器和/或分配器的对象。通过一个内部类来包装成员 `class _Impl : _Sp_ebo_helper<0, _Deleter>, _Sp_ebo_helper<1, _Alloc>`

    ```cpp
    template<typename _Ptr, typename _Deleter, typename _Alloc, _Lock_policy _Lp>
    class _Sp_counted_deleter final : public _Sp_counted_base<_Lp>
    {
        // 实现细节...
    private:
        _Ptr _M_ptr;           // 指向被管理对象的指针
        _Deleter _M_deleter;   // 自定义删除器
        _Alloc _M_alloc;       // 自定义分配器
    };
    ```

3. **`_Sp_counted_ptr_inplace`**：用于 `make_shared` 创建的对象，将对象直接构造在控制块内部。

   ```cpp
   template<typename _Tp, typename _Alloc, _Lock_policy _Lp>
   class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp>
   {
       class _Impl : _Sp_ebo_helper<0, _Alloc>
       {
       public:
           explicit _Impl(_Alloc __a) noexcept : _A_base(__a) { }
           _Alloc& _M_alloc() noexcept { return _A_base::_S_get(*this); }
           __gnu_cxx::__aligned_buffer<__remove_cv_t<_Tp>> _M_storage;
       };

       // ...

       _Tp* _M_ptr() noexcept { return _M_impl._M_storage._M_ptr(); }

       _Impl _M_impl;
   };
   ```

#### 创建规则

```cpp
// 1. 从原始指针构造，会创建一个新的控制块，通常是 `_Sp_counted_ptr` 类型
shared_ptr<T> sp(new T());

// 2. 从原始指针和自定义删除器构造，会创建一个 `_Sp_counted_deleter` 类型的控制块，存储自定义删除器
shared_ptr<T> sp(new T(), custom_deleter);

// 3. 使用 `make_shared` 构造，会创建一个 `_Sp_counted_ptr_inplace` 类型的控制块，对象直接构造在控制块内部。
auto sp = make_shared<T>();

// 4. 从另一个 `shared_ptr` 构造（拷贝构造），不会创建新的控制块，而是共享现有的控制块，并增加引用计数。
shared_ptr<T> sp2(sp1);

// 5. 从 `weak_ptr` 构造，如果 `weak_ptr` 没有过期（即其关联的对象仍然存在），则共享现有的控制块并增加引用计数；否则，抛出 `bad_weak_ptr` 异常。
shared_ptr<T> sp(wp);

// 6. 从 `unique_ptr` 构造，会创建一个新的控制块，通常是 `_Sp_counted_deleter` 类型，并接管 `unique_ptr` 的所有权。
shared_ptr<T> sp(std::move(up));
```

#### 内存管理

1. **分配**：
   - 对于普通的 `shared_ptr` 构造，控制块通常使用全局 `operator new` 分配在堆上。
   - 对于带有自定义分配器的构造，控制块使用提供的分配器分配内存。
   - 对于 `make_shared`，控制块和对象的内存一次性分配，通常使用全局 `operator new` 或提供的分配器。

2. **释放**：
   - 当强引用计数降为零时，对象被销毁（调用析构函数），但控制块可能仍然存在，如果有 `weak_ptr` 引用它。
   - 当弱引用计数也降为零时，控制块的内存被释放。
   - 对于 `make_shared` 创建的对象，即使对象已被销毁（强引用计数为零），但只要有 `weak_ptr` 引用控制块，对象占用的内存也不会被释放，因为对象和控制块共享同一块内存。

#### 线程安全

控制块的引用计数操作必须是线程安全的，以确保在多线程环境下正确工作。GCC 的 libstdc++ 实现提供了三种锁策略（`_Lock_policy`）：

1. **`_S_single`**：不使用任何同步机制，适用于单线程环境。

    ```cpp
    template<>
    inline void _Sp_counted_base<_S_single>::_M_add_ref_copy() { ++_M_use_count; }

    template<>
    inline void _Sp_counted_base<_S_single>::_M_release() noexcept {
      if (--_M_use_count == 0)
        {
          _M_dispose();
          if (--_M_weak_count == 0)
            _M_destroy();
        }
    }
    ```

2. **`_S_mutex`**：使用互斥锁进行同步，适用于不支持原子操作的平台。

    ```cpp
    template<>
    inline void _Sp_counted_base<_S_mutex>::_M_release() noexcept {
        // Be race-detector-friendly.  For more info see bits/c++config.
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_use_count);
        if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1) {
            _M_release_last_use();
        }
    }
    ```

3. **`_S_atomic`**：使用原子操作进行同步，这是现代平台上的默认选择。

    ```cpp
    template<>
    inline void _Sp_counted_base<_S_atomic>::_M_release() noexcept {
        if (__gnu_cxx::__exchange_and_add_dispatch(&_M_use_count, -1) == 1) {
            _M_release_last_use();
        }
    }
    ```

引用计数的增加和减少操作通常基于上面的原子操作实现，例如：

```cpp
// Increment the use count (used when the count is greater than zero).
void _M_add_ref_copy() { 
    __gnu_cxx::__atomic_add_dispatch(&_M_use_count, 1);
}

// Called by _M_release() when the use count reaches zero.
void _M_release_last_use() noexcept {
    _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_use_count);
    _M_dispose();           // 销毁对象
    // There must be a memory barrier between dispose() and destroy()
    // // to ensure that the effects of dispose() are observed in the
    // // thread that runs destroy().
    // // See http://gcc.gnu.org/ml/libstdc++/2005-11/msg00136.html
    if (_Mutex_base<_Lp>::_S_need_barriers) {
        __atomic_thread_fence (__ATOMIC_ACQ_REL);
    }

    // Be race-detector-friendly.  For more info see bits/c++config.
    _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_weak_count);
    if (__gnu_cxx::__exchange_and_add_dispatch(&_M_weak_count, -1) == 1) {
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_weak_count);
        _M_destroy();       // 销毁控制块
    }
}

// Increment the weak count.
void _M_weak_add_ref() noexcept { 
    __gnu_cxx::__atomic_add_dispatch(&_M_weak_count, 1); 
}

// Decrement the weak count.
void _M_weak_release() noexcept {
    // Be race-detector-friendly. For more info see bits/c++config.
    _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_weak_count);
    if (__gnu_cxx::__exchange_and_add_dispatch(&_M_weak_count, -1) == 1)
    {
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_weak_count);
        if (_Mutex_base<_Lp>::_S_need_barriers) {
        // See _M_release(),
        // destroy() must observe results of dispose()
        __atomic_thread_fence (__ATOMIC_ACQ_REL);
        }
        _M_destroy();
    }
}
```

#### 线程安全的边界

从上面的实现，我们可以梳理出 `shared_ptr` 线程安全的边界

1. **引用计数操作是线程安全的**：
   - 多个线程可以同时读取（复制）同一个 `shared_ptr` 对象
   - 多个线程可以同时销毁不同的 `shared_ptr` 对象，即使它们指向相同的资源
   - 引用计数的增加和减少操作是原子的，不会导致数据竞争

2. **对象访问不是线程安全的**：
   - `shared_ptr` 不保证其指向的对象的线程安全性
   - 如果多个线程通过 `shared_ptr` 访问同一个对象，需要额外的同步机制

3. **单个 `shared_ptr` 对象的修改不是线程安全的**：
   - 多个线程不能同时修改同一个 `shared_ptr` 对象（例如，通过赋值或 `reset()`）
   - 对同一个 `shared_ptr` 对象的并发读写也不是线程安全的
  
#### 内存序的应用

1. **引用计数的增加**：通常使用 `memory_order_relaxed` 或 `memory_order_acq_rel`，因为这个操作只需要保证计数的原子性，不需要额外的内存屏障。

2. **引用计数的减少**：通常使用更强的内存序，如 `memory_order_acq_rel` 或 `memory_order_release`，特别是当引用计数降为零时，需要确保之前的所有操作对后续的销毁操作可见。

3. **`use_count()` 方法**：通常使用 `memory_order_relaxed`，因为这个方法主要用于调试，不需要强同步保证。

   ```cpp
   long _M_get_use_count() const noexcept
   {
       // 使用 relaxed 内存序读取引用计数
       return __atomic_load_n(&_M_use_count, __ATOMIC_RELAXED);
   }
   ```

#### 线程安全性相关的实现细节

`shared_ptr` 的线程安全实现涉及一些关键的技术细节：

1. **原子引用计数**：
   - 使用原子类型（如 `_Atomic_word`）存储引用计数
   - 使用原子操作（如 `__atomic_add_dispatch`）修改引用计数
   - 使用适当的内存序确保操作的可见性

2. **内存屏障**：
   - 在某些关键操作（如销毁对象）前后插入内存屏障
   - 确保对象的销毁在所有使用该对象的线程可见

    ```cpp
    // Called by _M_release() when the use count reaches zero.
    void _M_release_last_use() noexcept {
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_use_count);
        _M_dispose();
        // There must be a memory barrier between dispose() and destroy()
        // to ensure that the effects of dispose() are observed in the
        // thread that runs destroy().
        // See http://gcc.gnu.org/ml/libstdc++/2005-11/msg00136.html
        if (_Mutex_base<_Lp>::_S_need_barriers) {
            __atomic_thread_fence (__ATOMIC_ACQ_REL);
        }

        // Be race-detector-friendly.  For more info see bits/c++config.
        _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_weak_count);
        if (__gnu_cxx::__exchange_and_add_dispatch(&_M_weak_count, -1) == 1) {
            _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_weak_count);
            _M_destroy();
        }
    }
    ```

3. **线程安全的弱引用**：
   - `weak_ptr` 的引用计数操作也是线程安全的
   - `weak_ptr::lock()` 方法使用原子操作检查和增加引用计数

    ```cpp
    template<>
    inline bool _Sp_counted_base<_S_atomic>::_M_add_ref_lock_nothrow() noexcept {
        // Perform lock-free add-if-not-zero operation.
        _Atomic_word __count = _M_get_use_count();
        do {
            if (__count == 0)
                return false;
            // Replace the current counter value with the old value + 1, as
            // long as it's not changed meanwhile.
        } while (!__atomic_compare_exchange_n(&_M_use_count, &__count, __count + 1,
                                              true, __ATOMIC_ACQ_REL, __ATOMIC_RELAXED));
        return true;
    }
    ```

### C++20 中的 [`std::atomic<std::shared_ptr>`](https://en.cppreference.com/w/cpp/memory/shared_ptr/atomic2)

> The partial template specialization of std::atomic for std::shared_ptr<T> allows users to manipulate shared_ptr objects atomically.
> 
> If multiple threads of execution access the same std::shared_ptr object without synchronization and any of those accesses uses a non-const member function of shared_ptr then a data race will occur unless all such access is performed through an instance of std::atomic<std::shared_ptr> (or, deprecated as of C++20, through the standalone functions for atomic access to std::shared_ptr).
> 
> Associated use_count increments are guaranteed to be part of the atomic operation. Associated use_count decrements are sequenced after the atomic operation, but are not required to be part of it, except for the use_count change when overriding expected in a failed CAS. Any associated deletion and deallocation are sequenced after the atomic update step and are not part of the atomic operation.
> 
> Note that the control block of a shared_ptr is thread-safe: different non-atomic std::shared_ptr objects can be accessed using mutable operations, such as operator= or reset, simultaneously by multiple threads, even when these instances are copies, and share the same control block internally.

在 GCC 的 libstdc++ 实现中，`std::atomic<std::shared_ptr>` 的核心是 `_Sp_atomic` 类：

```cpp
// gcc/libstdc++-v3/include/bits/shared_ptr_atomic.h
template<typename _Tp>
class _Sp_atomic
{
    // ...
private:
    typename _Tp::element_type* _M_ptr = nullptr;
    _Atomic_count _M_refcount;
    // ...
};
```

`_Sp_atomic` 类使用一个特殊的 `_Atomic_count` 类型来管理引用计数，该类型使用原子操作和锁来确保线程安全：

```cpp
// An atomic version of __shared_count<> and __weak_count<>.
// Stores a _Sp_counted_base<>* but uses the LSB as a lock.
struct _Atomic_count {
    ……
    
    // Precondition: Caller does not hold lock!
    // Returns the raw pointer value without the lock bit set.
    pointer lock(memory_order __o) const noexcept {
        // To acquire the lock we flip the LSB from 0 to 1.
        auto __current = _M_val.load(memory_order_relaxed);
        while (__current & _S_lock_bit) {
#if __glibcxx_atomic_wait
            __detail::__thread_relax();
#endif
            __current = _M_val.load(memory_order_relaxed);
        }

        _GLIBCXX_TSAN_MUTEX_TRY_LOCK(&_M_val);

        while (!_M_val.compare_exchange_strong(__current, __current | _S_lock_bit,
                                               __o, memory_order_relaxed)) {
            _GLIBCXX_TSAN_MUTEX_TRY_LOCK_FAILED(&_M_val);
#if __glibcxx_atomic_wait
            __detail::__thread_relax();
#endif
            __current = __current & ~_S_lock_bit;
            _GLIBCXX_TSAN_MUTEX_TRY_LOCK(&_M_val);
        }
        _GLIBCXX_TSAN_MUTEX_LOCKED(&_M_val);
        return reinterpret_cast<pointer>(__current);
    }

    // Precondition: caller holds lock!
    void unlock(memory_order __o) const noexcept {
      _GLIBCXX_TSAN_MUTEX_PRE_UNLOCK(&_M_val);
      _M_val.fetch_sub(1, __o);
      _GLIBCXX_TSAN_MUTEX_POST_UNLOCK(&_M_val);
    }
};

```

### `make_shared` 的实现原理

我们追踪的 `make_shared` 的一条调用链路如下：

```cpp
/**
 *  @brief  Create an object that is owned by a shared_ptr.
 *  @param  __args  Arguments for the @a _Tp object's constructor.
 *  @return A shared_ptr that owns the newly created object.
 *  @throw  std::bad_alloc, or an exception thrown from the
 *          constructor of @a _Tp.
 */
template<typename _Tp, typename... _Args>
inline shared_ptr<_NonArray<_Tp>> make_shared(_Args&&... __args) {
    using _Alloc = allocator<void>;
    _Alloc __a;
    return shared_ptr<_Tp>(_Sp_alloc_shared_tag<_Alloc>{__a}, std::forward<_Args>(__args)...);
}

template<typename _Alloc, typename... _Args>
shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args) : __shared_ptr<_Tp>(__tag, std::forward<_Args>(__args)...) {}

template<typename _Alloc, typename... _Args>
__shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args) : _M_ptr(), _M_refcount(_M_ptr, __tag, std::forward<_Args>(__args)...) { 
    _M_enable_shared_from_this_with(_M_ptr); 
}

template<typename _Tp, typename _Alloc, typename... _Args>
__shared_count(_Tp*& __p, _Sp_alloc_shared_tag<_Alloc> __a, _Args&&... __args) {
    using _Tp2 = __remove_cv_t<_Tp>;
    using _Sp_cp_type = _Sp_counted_ptr_inplace<_Tp2, _Alloc, _Lp>;
    typename _Sp_cp_type::__allocator_type __a2(__a._M_a);
    auto __guard = std::__allocate_guarded(__a2);
    _Sp_cp_type* __mem = __guard.get();
    auto __pi = ::new (__mem)
    _Sp_cp_type(__a._M_a, std::forward<_Args>(__args)...);
    __guard = nullptr;
    _M_pi = __pi;
    __p = __pi->_M_ptr();
}
```

这里的关键是 `_Sp_counted_ptr_inplace` 类型，在前面控制块的类型中我们也有提到。它是一个特殊的控制块类型，用于 `make_shared` 创建的对象。与普通控制块不同，它将对象直接构造在控制块内部，而不是在外部分配。

```cpp
template<typename _Tp, typename _Alloc, _Lock_policy _Lp>
class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp>
{
    class _Impl : _Sp_ebo_helper<0, _Alloc>
    {
    public:
        explicit _Impl(_Alloc __a) noexcept : _A_base(__a) { }
        _Alloc& _M_alloc() noexcept { return _A_base::_S_get(*this); }
        __gnu_cxx::__aligned_buffer<__remove_cv_t<_Tp>> _M_storage;
    };
    // ...
    _Tp* _M_ptr() noexcept { return _M_impl._M_storage._M_ptr(); }
    _Impl _M_impl;
};
```

在这个实现中，`_M_storage` 是一个对齐的内存缓冲区，用于存储对象。当 `make_shared` 被调用时，它会分配足够的内存来容纳控制块和对象，然后在这个内存中构造对象。

C++20 添加了对数组的支持，允许使用 `make_shared` 创建动态数组。

```cpp
// C++20
auto sp = std::make_shared<int[]>(10);  // 创建一个包含 10 个 int 的数组
auto sp2 = std::make_shared<int[10]>();  // 创建一个包含 10 个 int 的数组
```

C++20 添加了 `make_shared_for_overwrite`，它创建一个未初始化的对象，类似于 `new T`（而不是 `new T()`）。

```cpp
// C++20
auto sp = std::make_shared_for_overwrite<MyClass>();
// 相当于 shared_ptr<MyClass>(new MyClass)，而不是 shared_ptr<MyClass>(new MyClass())
```

## 一些值得思考的问题

### `shard_ptr`构造和`make_shared`的区别

1. **使用 `new` 表达式和 `shared_ptr` 构造函数**：

```cpp
std::shared_ptr<T> sp(new T());
```

在这种情况下，对象和控制块在内存中是分开分配的。首先通过 `new T()` 在堆上分配对象，然后 `shared_ptr` 构造函数会在堆上分配一个单独的控制块。这涉及到两次内存分配操作。

```text
                            +--------------+
                    +-----> |     对象      |  (单独分配)
+-------------+     |       +--------------+
| shared_ptr  |     |
| _M_ptr      |-----+       +--------------+
| _M_refcount |---------->  |     控制块    |   (单独分配)
+-------------+             +--------------+
```

2. **使用 `std::make_shared`**：

```cpp
auto sp = std::make_shared<T>();
```

在这种情况下，`make_shared` 会进行单次内存分配，同时为对象和控制块分配内存。对象通常直接构造在控制块之后的内存区域中，形成一个连续的内存块。这种方法不仅减少了内存分配的次数，还提高了缓存局部性。

```text          
+-------------+             +--------------+
| shared_ptr  |     +-----> |      对象     | <-+
| _M_ptr      |-----+       +--------------+   |
| _M_refcount |---------->  |     控制块    | --+   
+-------------+             +--------------+
```

详细的解释可以参考 [std::make_shared](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared)，部分关键描述摘录如下：

> These functions will typically allocate more memory than sizeof(T) to allow for internal bookkeeping structures such as reference counts.
> 
> These functions may be used as an alternative to std::shared_ptr<T>(new T(args...)). The trade-offs are:
>
> * std::shared_ptr<T>(new T(args...)) performs at least two allocations (one for the object T and one for the control block of the shared pointer), while std::make_shared<T> typically performs only one allocation (the standard recommends, but does not require this; all known implementations do this).
>
> * Unlike the std::shared_ptr constructors, std::make_shared does not allow a custom deleter.

同时`make_shared` 还提供了更好的异常安全性。考虑以下代码：

```cpp
void foo(std::shared_ptr<MyClass> p1, std::shared_ptr<MyClass> p2);

// 方式 1
foo(std::shared_ptr<MyClass>(new MyClass()), std::shared_ptr<MyClass>(new MyClass()));

// 方式 2
foo(std::make_shared<MyClass>(), std::make_shared<MyClass>());
```

在方式 1 中，编译器可能按以下顺序执行操作：

1. 分配第一个 `MyClass` 对象的内存
2. 分配第二个 `MyClass` 对象的内存
3. 构造第一个 `MyClass` 对象
4. 构造第二个 `MyClass` 对象
5. 分配第一个控制块的内存
6. 分配第二个控制块的内存
7. 构造第一个 `shared_ptr`
8. 构造第二个 `shared_ptr`

如果在步骤 6 中发生异常（例如，内存不足），那么第一个 `MyClass` 对象的内存将泄漏，因为它还没有被 `shared_ptr` 管理。

而在方式 2 中，每个 `make_shared` 调用都是原子的，不会出现这种内存泄漏的情况。即使第二个 `make_shared` 调用失败，第一个调用创建的对象也已经被 `shared_ptr` 正确管理。

当然 `make_shared` 也并非全是优点，这里需要额外引起注意的是：

- 在 `make_shared` 构造的情况下，`shared_ptr` 自身的内存和用户分配的内存在一起，所以就不能通过传 `deleter` 的方式实现自定义析构了。
- 由于对象和控制块在同一块内存中，只有当最后一个 `shared_ptr` 和最后一个 `weak_ptr` 都被销毁时，内存才会被释放。这意味着，即使对象不再需要（所有 `shared_ptr` 都已销毁），只要还有 `weak_ptr` 引用控制块，对象占用的内存也不会被释放。
- 在某些情况下，`make_shared` 可能无法满足特殊的对齐要求，特别是当对象需要比默认对齐更严格的对齐时。
- 如果类的构造函数是私有的或受保护的，并且 `make_shared` 不是友元，那么无法使用 `make_shared` 创建该类的对象。


### 控制块内的指针和别名构造（Aliasing Constructor）

前面提到 `shared_ptr` 自身持有和其内部控制块持有的指针可能是不同类型的，这有什么用处呢？

标准库为 `shared_ptr` 提供了一个叫做别名构造的能力，参考 [std::shared_ptr<T>::shared_ptr(8)](https://en.cppreference.com/w/cpp/memory/shared_ptr/shared_ptr.html)

```cpp
template< class Y >
shared_ptr( const shared_ptr<Y>& r, element_type* ptr ) noexcept;   

template< class Y >
shared_ptr( shared_ptr<Y>&& r, element_type* ptr ) noexcept;            // (since C++ 20)
```

对它的原文解释如下：

> 8) The aliasing constructor: constructs a shared_ptr which shares ownership information with the initial value of r, but holds an unrelated and unmanaged pointer ptr. If this shared_ptr is the last of the group to go out of scope, it will call the stored deleter for the object originally managed by r. However, calling get() on this shared_ptr will always return a copy of ptr. It is the responsibility of the programmer to make sure that this ptr remains valid as long as this shared_ptr exists, such as in the typical use cases where ptr is a member of the object managed by r or is an alias (e.g., downcast) of r.get() For the second overload taking an rvalue, r is empty and r.get() == nullptr after the call.(since C++20)

翻译翻译：

* 别名构造可以让 `shared_ptr` 保存两个指针，一个通过引用计数管理的指针 `r`，一个无关且不受控制块管理的指针 `ptr`
* `shared_ptr` 不会为 `ptr` 创建控制块
* `get()` 方法会返回 `ptr` 的拷贝，不再返回 `r`
* 调用者负责保证 `ptr` 在 `r` 的生命周期上一直有效

同时文中还建议我们，推荐的用法是 `ptr` 是 `r` 的一个成员或别名，比如下面的用法：

```cpp
   struct Foo { int x; };
   std::shared_ptr<Foo> p1(new Foo);
   std::shared_ptr<int> p2(p1, &p1->x);  // p2 指向 p1->x，但共享 p1 的控制块
```

使用两个指针，还会带来一些其他的好处：

**支持自定义删除器和分配器**：控制块需要存储自定义删除器和分配器，这些不能直接与对象指针合并。通过将控制块与对象指针分离，可以灵活地支持各种删除器和分配器，而不影响对象的访问。

**类型擦除（Type Erasure）**：控制块包含了删除器的类型擦除版本，这使得不同类型的删除器可以与同一个 `shared_ptr` 类型一起使用。这种类型擦除需要额外的内存和间接性，因此将其放在单独的控制块中是合理的。

同时控制块的分离存储也是有意为之：

**共享所有权**：控制块的核心目的是实现共享所有权。多个 `shared_ptr` 实例需要共享同一个控制块，以便正确跟踪引用计数。如果控制块嵌入在 `shared_ptr` 中，每个 `shared_ptr` 都会有自己的引用计数，无法实现共享所有权。

**内存效率**：分离控制块允许多个 `shared_ptr` 实例共享同一个控制块，而不需要为每个实例复制控制块的内容。这减少了内存使用，特别是当有大量 `shared_ptr` 指向同一个对象时。

**支持 `weak_ptr`**：`weak_ptr` 需要访问控制块以检查对象是否仍然存在，但不增加强引用计数。如果控制块嵌入在 `shared_ptr` 中，当最后一个 `shared_ptr` 被销毁时，控制块也会被销毁，`weak_ptr` 将无法正常工作。

**线程安全性**：控制块包含需要原子操作的引用计数。将这些操作集中在一个地方（控制块）简化了线程安全性的实现，并减少了对 `shared_ptr` 对象本身的同步要求。

### `enable_shared_from_this`

`std::enable_shared_from_this` 是一个模板基类，允许对象从 `this` 指针安全地创建 `shared_ptr`。这解决了一个常见问题：如何从对象的成员函数中获取指向自身的 `shared_ptr`。

没有 `enable_shared_from_this`，用户可能会尝试直接从 `this` 创建 `shared_ptr`：

```cpp
class Bad {
public:
    std::shared_ptr<Bad> getptr() {
        return std::shared_ptr<Bad>(this);  // 危险！会导致双重释放
    }
};
```

这是危险的，因为它会创建一个新的控制块，导致同一个对象被多个独立的控制块管理，最终导致双重释放。

`enable_shared_from_this` 通过以下机制解决这个问题：

1. **存储弱引用**：
   
   `enable_shared_from_this` 在对象内部存储一个指向自身的 `weak_ptr`。

2. **自动初始化**：
   
   当第一个管理对象的 `shared_ptr` 被创建时，它会自动初始化 `enable_shared_from_this` 中的弱引用。

3. **安全创建**：
   
   `shared_from_this()` 方法使用存储的弱引用创建一个新的 `shared_ptr`，确保它共享现有的控制块，而不是创建新的。

这种设计允许对象安全地参与自己的引用计数管理，而不会导致双重释放问题。

## 最佳实践

1. **避免对同一个 `shared_ptr` 对象的并发修改**：
   - 在 C++20 之前，需要使用互斥锁保护对同一个 `shared_ptr` 对象的并发修改
   - 在 C++20 及以后，可以使用 `std::atomic<std::shared_ptr>`
2. **避免从原始指针创建多个 `shared_ptr`**：
   - 这会创建多个独立的控制块，导致双重释放
   - 应该使用现有的 `shared_ptr` 或 `weak_ptr` 创建新的 `shared_ptr`
3. **使用 `make_shared` 减少竞态条件**：
   - `make_shared` 在单个原子操作中分配对象和控制块，减少了竞态条件的可能性
4. **注意 `use_count()` 的非确定性**：
   - 在多线程环境中，`use_count()` 返回的值可能是不准确的
   - 不应该依赖 `use_count()` 的精确值做重要决策
5. **使用 `weak_ptr` 打破循环引用**：
   - 循环引用会导致内存泄漏，即使在多线程环境中也是如此
   - 使用 `weak_ptr` 可以安全地打破循环引用
6. **避免不必要的拷贝**：每次拷贝 `shared_ptr` 都会导致引用计数的原子增加操作，这在高频调用路径上可能成为性能瓶颈。
7. **使用自定义删除器时的注意事项**：自定义删除器会增加控制块的大小，特别是当删除器是函数对象且包含状态时。为了最小化这种开销：

   ```cpp
   // 不推荐：有状态函数对象作为删除器
   struct MyDeleterWithState {
       int state;
       void operator()(Widget* w) const { delete w; }
   };
   std::shared_ptr<Widget> sp(new Widget, MyDeleterWithState{42});
   
   // 推荐：无状态 lambda 作为删除器
   auto sp = std::shared_ptr<Widget>(new Widget, [](Widget* w) { delete w; });

   // 对于频繁使用的删除器，考虑使用函数指针。函数指针通常比函数对象占用更少的空间
   void delete_widget(Widget* w) { delete w; }
   std::shared_ptr<Widget> sp(new Widget, delete_widget);
   ```

8. **`enable_shared_from_this` 的正确使用**

    ```cpp
    class Widget : public std::enable_shared_from_this<Widget> {
    public:
        Widget() {
            // 错误：在构造函数中调用 shared_from_this()
            auto self = shared_from_this();  // 抛出 std::bad_weak_ptr 异常
        }

        void doSomething() {
            // 错误：对象不是由 shared_ptr 管理的
            auto self = shared_from_this();  // 如果对象不是由 shared_ptr 管理，抛出异常
        }
    };

    void problem() {
        Widget* w = new Widget();  // 直接创建，不是由 shared_ptr 管理
        w->doSomething();  // 将抛出异常
        delete w;
    }
    ```

## 引用

- [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr.html)
- [shared_ptr.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr.h)
- [shared_ptr_atomic.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_atomic.h)
- [shared_ptr_base.h](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/shared_ptr_base.h)
