# 详解[`std::variant`](https://en.cppreference.com/w/cpp/utility/variant.html)

`std::variant` 是C++17标准库中引入的一个重要组件，它提供了一种类型安全的联合体（union）实现。在C++17之前，开发者需要使用传统的C风格联合体、继承多态或第三方库（如Boost.Variant）来实现类似的功能，但这些方案都有各自的局限性。

传统的C风格联合体虽然高效，但存在严重的类型安全问题，不能直接存储具有非平凡构造函数或析构函数的类型，也无法自动跟踪当前存储的类型。

而基于继承的多态虽然类型安全，但要求所有类型必须有共同基类，且通常需要动态内存分配，带来了额外的性能开销。

`std::variant` 的设计目标是提供一个既类型安全又高效的解决方案，能够在不需要继承关系的情况下实现值语义的多态行为。

`std::variant` 是C++17标准的一部分，与 `std::optional` 和 `std::any` 一起被引入，这三个组件共同构成了C++17中的"变体类型"系列。

## 基本概念

类模板`std::variant`表示一种**类型安全的联合体**。具有以下特点和约束：

1. 任意时刻，`variant`的实例要么持有其备选类型之一的值，或因错误处于无值状态（极少见，参见`valueless_by_exception`）；
2. 与联合体（union）类似，若`variant`持有某对象类型`T`的值，则该`T`对象会嵌套在`variant`对象内部。
3. `variant`禁止持有引用、数组、`void`；允许重复持有同一类型（多次持有同一类型）或其不同cv限定版本（`const`、`volatile`）；
4. 通过默认构造的`variant`会持有其**第一个备选类型**的值，除非该备选类型不可默认构造（此时`variant`自身也不可默认构造）。辅助类`std::monostate`可用于让这类`variant`变得可默认构造。
5. 声明`variant`必须传入至少一个模板参数（无参数`std::variant<>`非法），无参数场景用`std::variant<std::monostate>`替代。
6. 程序不得声明`std::variant`的显式特化或部分特化（如`template <> class std::variant<int>`或`template <class T> class std::variant<T, int>`），这种用法是错误的（ill-formed）。

## 接口和用法

## 源码分析

> gcc           libstdc++-v3/include/std/variant
> llvm-libcxx   libcxx/include/variant

我们先以分析`releases/gcc-15.2.0`的实现（位于`libstdc++-v3/include/std/variant`），然后再回头看看llvm的实现有什么区别。

### 继承体系

首先我们来分析一下 `std::variant` 的继承体系。

```cpp
template<typename... _Types>
class variant : private __detail::__variant::_Variant_base<_Types...>,          // 核心的存储结构
                private _Enable_copy_move<                                      // 控制 复制、移动 函数的启用
                    __detail::__variant::_Traits<_Types...>::_S_copy_ctor,
                    __detail::__variant::_Traits<_Types...>::_S_copy_assign,
                    __detail::__variant::_Traits<_Types...>::_S_move_ctor,
                    __detail::__variant::_Traits<_Types...>::_S_move_assign,
                    variant<_Types...>> {
    using _Base = __detail::__variant::_Variant_base<_Types...>;
    using _Traits = __detail::__variant::_Traits<_Types...>;

    ……
};
```

`variant`直接继承了两个父类： **`__detail::__variant::_Variant_base<_Types...>`**，**`_Enable_copy_move<...>`**

#### `_Variant_base`继承链

继续深入的话，可以看到 `std::variant` 具有一个比较长的继承链，从 `_Variant_base` 一直到最内层的 `_Variant_storage`:

`_Variant_base <- _Move_assign_alias <- _Move_assign_base <- _Copy_assign_alias <- _Copy_assign_base <- _Move_ctor_alias <- _Move_ctor_base <- _Copy_ctor_alias <- _Copy_ctor_base <- _Variant_storage_alias <- _Variant_storage`

这么设计的目的是**根据备选类型的特性（如是否平凡析构、复制、移动），条件性地实现存储管理、构造和赋值逻辑**

由内而外的逐级类型如下：

**1. 最内层的Union**

`_Variadic_union` 是一个嵌套递归的 `union`，它不在`variant`的继承链上，而是作为 `_Variant_storage` 的成员变量，完成实际的存储功能

```cpp
// 嵌套递归的 union，本质上和一个平铺的 union 是一样的，
// 但通过 _Uninitialized 的特化，解决了类型是否支持平凡析构的区别
template <bool __trivially_destructible, typename _First, typename... _Rest>
union _Variadic_union<__trivially_destructible, _First, _Rest...> {
  _Uninitialized<_First> _M_first;                              // 第一个类型的存储
  _Variadic_union<__trivially_destructible, _Rest...> _M_rest;  // 剩余类型的递归存储
};

/** _Uninitialized 具有两个特化，针对是否可以平凡析构 **/
// 平凡析构类型：直接存储对象
template <typename _Type, true> struct _Uninitialized {
  _Type _M_storage;  // 直接存储对象

  template <typename... _Args>
  constexpr _Uninitialized(in_place_index_t<0>, _Args &&...__args)
      : _M_storage(std::forward<_Args>(__args)...) {}   // 原地构造
};

// 非平凡析构类型：使用一块对齐的缓冲区
template <typename _Type> struct _Uninitialized<_Type, false> {
  __gnu_cxx::__aligned_membuf<_Type> _M_storage;    // 内存对齐的缓冲区
  // placement new
  template <typename... _Args>
  constexpr _Uninitialized(in_place_index_t<0>, _Args &&...__args) {
    ::new ((void *)std::addressof(_M_storage)) _Type(std::forward<_Args>(__args)...);
  }
};
```

**2. `_Variant_storage`——存储与索引管理**

`_Variant_storage`持有上面的`Union`，直接管理`variant`的内存存储和活跃类型索引

```cpp
template <bool __trivially_destructible, typename... _Types>
struct _Variant_storage {
  _Variadic_union<__trivially_destructible, _Types...> _M_u;  // 存储备选类型的联合体
  using __index_type = __select_index<_Types...>;  // 最小化的索引类型（如char/short）
  __index_type _M_index;  // 记录当前活跃类型的索引（variant_npos表示无值）

  // 销毁当前活跃类型（非平凡析构时调用）
  constexpr void _M_reset() { /* ... */ }

  // 检查是否有有效值（非valueless状态）
  constexpr bool _M_valid() const noexcept { /* ... */ }
};
```

**3. `_Copy_ctor_base`——复制构造**

`_Copy_ctor_base`用于实现复制构造函数，根据备选类型是否均为**平凡复制构造**（`is_trivially_copy_constructible`）来实例化：

```cpp
// 非平凡复制构造的实现（当_Traits::_S_trivial_copy_ctor为false时）
template <bool, typename... _Types>
struct _Copy_ctor_base : _Variant_storage_alias<_Types...> {
  using _Base = _Variant_storage_alias<_Types...>;

  _GLIBCXX20_CONSTEXPR
  _Copy_ctor_base(const _Copy_ctor_base& __rhs) noexcept(...) {
    // 遍历__rhs的活跃类型，复制构造到当前对象
    __variant::__raw_idx_visit(
        [this](auto&& __rhs_mem, auto __rhs_index) {
            constexpr size_t __j = __rhs_index;
            if constexpr (__j != variant_npos)
                std::_Construct(std::__addressof(this->_M_u), in_place_index<__j>, __rhs_mem);
        }, 
        __variant_cast<_Types...>(__rhs));
    this->_M_index = __rhs._M_index;
  }
};

// 平凡复制构造的实现（当_Traits::_S_trivial_copy_ctor为true时）
template <typename... _Types>
struct _Copy_ctor_base<true, _Types...> : _Variant_storage_alias<_Types...> {
  using _Base = _Variant_storage_alias<_Types...>;
  using _Base::_Base;  // 直接使用编译器生成的默认复制构造
};
```

**4. `_Move_ctor_base`——条件性移动构造**

与`_Copy_ctor_base`类似，`_Move_ctor_base`根据备选类型是否均为**平凡移动构造**（`is_trivially_move_constructible`）来实例化：

```cpp
// 非平凡移动构造的实现（当_Traits::_S_trivial_move_ctor为false时）
template <bool, typename... _Types>
struct _Move_ctor_base : _Copy_ctor_alias<_Types...> {
  using _Base = _Copy_ctor_alias<_Types...>;

  _GLIBCXX20_CONSTEXPR
  _Move_ctor_base(_Move_ctor_base&& __rhs) noexcept(...) {
    // 遍历__rhs的活跃类型，移动构造到当前对象
    __variant::__raw_idx_visit(
        [this](auto &&__rhs_mem, auto __rhs_index) mutable {
          constexpr size_t __j = __rhs_index;
          if constexpr (__j != variant_npos)
            std::_Construct(std::__addressof(this->_M_u), in_place_index<__j>,
                            std::forward<decltype(__rhs_mem)>(__rhs_mem));
        },
        __variant_cast<_Types...>(std::move(__rhs)));
    this->_M_index = __rhs._M_index;
  }
};

// 平凡移动构造的实现（当_Traits::_S_trivial_move_ctor为true时）
template <typename... _Types>
struct _Move_ctor_base<true, _Types...> : _Copy_ctor_alias<_Types...> {
  using _Base = _Copy_ctor_alias<_Types...>;
  using _Base::_Base;  // 直接使用默认移动构造
};
```

**5. `_Copy_assign_base`——条件性复制赋值**

`_Copy_assign_base`处理复制赋值操作，根据备选类型是否均为**平凡复制赋值**（`is_trivially_copy_assignable`）来实例化：

```cpp
// 非平凡复制赋值的实现（当_Traits::_S_trivial_copy_assign为false时）
template <bool, typename... _Types>
struct _Copy_assign_base : _Move_ctor_alias<_Types...> {
  using _Base = _Move_ctor_alias<_Types...>;

  _GLIBCXX20_CONSTEXPR
  _Copy_assign_base& operator=(const _Copy_assign_base& __rhs) noexcept(...) {
    // 分情况处理：右值无值、同类型赋值、不同类型赋值
    __variant::__raw_idx_visit(
        [this](auto &&__rhs_mem, auto __rhs_index) mutable {
          constexpr size_t __j = __rhs_index;
          if constexpr (__j == variant_npos)
            this->_M_reset();                           // 右值无值，当前对象也置为无值
          else if (this->_M_index == __j)
            __variant::__get<__j>(*this) = __rhs_mem;   // 同类型，直接赋值
          else {                                        // 不同类型，销毁当前类型并复制构造新类型
            using _Tj = typename _Nth_type<__j, _Types...>::type;
            if constexpr (is_nothrow_copy_constructible_v<_Tj> ||
                          !is_nothrow_move_constructible_v<_Tj>)
              __variant::__emplace<__j>(*this, __rhs_mem);  
            else {
              using _Variant = variant<_Types...>;
              _Variant &__self = __variant_cast<_Types...>(*this);
              __self = _Variant(in_place_index<__j>, __rhs_mem);
            }
          }
        },
        __variant_cast<_Types...>(__rhs));
    return *this;
  }
};

// 平凡复制赋值的实现（当_Traits::_S_trivial_copy_assign为true时）
template <typename... _Types>
struct _Copy_assign_base<true, _Types...> : _Move_ctor_alias<_Types...> {
  using _Base = _Move_ctor_alias<_Types...>;
  using _Base::operator=;  // 直接使用默认复制赋值
};
```

**6. `_Move_assign_base`——条件性移动赋值**

`_Move_assign_base`处理移动赋值操作，逻辑与复制赋值类似，但针对移动语义优化：

```cpp
// 非平凡移动赋值的实现（当_Traits::_S_trivial_move_assign为false时）
template <bool, typename... _Types>
struct _Move_assign_base : _Copy_assign_alias<_Types...> {
  using _Base = _Copy_assign_alias<_Types...>;

  _GLIBCXX20_CONSTEXPR
  _Move_assign_base& operator=(_Move_assign_base&& __rhs) noexcept(...) {
    // 分情况处理：右值无值、同类型赋值、不同类型赋值
    __variant::__raw_idx_visit(
        [this](auto &&__rhs_mem, auto __rhs_index) mutable {
          constexpr size_t __j = __rhs_index;
          if constexpr (__j != variant_npos) {
            if (this->_M_index == __j)
              __variant::__get<__j>(*this) = std::move(__rhs_mem);  // 同类型，移动赋值
            else {                                                  // 不同类型，销毁当前类型并移动构造新类型
              using _Tj = typename _Nth_type<__j, _Types...>::type; // 从模板参数包`_Types...`中提取第`__j`个位置的类型
              if constexpr (is_nothrow_move_constructible_v<_Tj>)
                // 若`_Tj`是无异常复制构造，则直接通过`__emplace<__j>`在当前`variant`中销毁旧类型并构造`_Tj`的副本
                __variant::__emplace<__j>(*this, std::move(__rhs_mem));
              else {
                // 否则，通过临时`variant`对象先构造新类型，再移动赋值给当前`variant`
                // 保证异常安全（若构造临时对象时抛出异常，当前`variant`状态不受影响）
                using _Variant = variant<_Types...>;
                _Variant &__self = __variant_cast<_Types...>(*this);
                __self.template emplace<__j>(std::move(__rhs_mem));
              }
            }
          } else
            this->_M_reset();
        },
        __variant_cast<_Types...>(__rhs));
    return *this;
  }
};

// 平凡移动赋值的实现（当_Traits::_S_trivial_move_assign为true时）
template <typename... _Types>
struct _Move_assign_base<true, _Types...> : _Copy_assign_alias<_Types...> {
  using _Base = _Copy_assign_alias<_Types...>;
  using _Base::operator=;  // 直接使用默认移动赋值
};
```

**7. 顶层：`_Variant_base`**

`_Variant_base`是继承链的顶层，直接被`variant`继承，负责统一构造函数接口：

```cpp
template <typename... _Types>
struct _Variant_base : _Move_assign_alias<_Types...> {
  using _Base = _Move_assign_alias<_Types...>;

  // 默认构造函数：初始化第一个备选类型（要求其可默认构造）
  constexpr _Variant_base() noexcept(...) : _Variant_base(in_place_index<0>) {}

  // 带索引的构造函数：根据索引构造指定类型
  template <size_t _Np, typename... _Args>
  constexpr explicit _Variant_base(in_place_index_t<_Np> __i, _Args&&... __args)
      : _Base(__i, std::forward<_Args>(__args)...) {}

  // 继承所有复制/移动构造和赋值
  _Variant_base(const _Variant_base&) = default;
  _Variant_base(_Variant_base&&) = default;
  _Variant_base& operator=(const _Variant_base&) = default;
  _Variant_base& operator=(const _Variant_base&&) = default;
};
```

#### `_Enable_copy_move`控制成员函数的生成

`_Enable_copy_move`是GNU标准库的内部工具类，用于**根据类型特性条件性地启用或禁用复制/移动函数**，保证`variant`仅在备选类型均支持复制/移动时，才对外提供对应的复制/移动接口，位于`libstdc++-v3/include/bits/enable_special_members.h`：

```cpp
/**
  * @brief A mixin helper to conditionally enable or disable the default
  * destructor.
  * @sa _Enable_special_members
  */
template<bool _Switch, typename _Tag = void>
  struct _Enable_destructor { };

/**
  * @brief A mixin helper to conditionally enable or disable the copy/move
  * special members.
  * @sa _Enable_special_members
  */
template<bool _Copy, bool _CopyAssignment,
         bool _Move, bool _MoveAssignment,
         typename _Tag = void>
  struct _Enable_copy_move { };

/**
  * @brief A mixin helper to conditionally enable or disable the special
  * members.
  *
  * The @c _Tag type parameter is to make mixin bases unique and thus avoid
  * ambiguities.
  */
template<bool _Default, bool _Destructor,
         bool _Copy, bool _CopyAssignment,
         bool _Move, bool _MoveAssignment,
         typename _Tag = void>
  struct _Enable_special_members
  : private _Enable_default_constructor<_Default, _Tag>,
    private _Enable_destructor<_Destructor, _Tag>,
    private _Enable_copy_move<_Copy, _CopyAssignment,
                              _Move, _MoveAssignment,
                              _Tag>
  { };


// 标准库在这里穷举了这个模板类的偏特化，下面是其中一个示例
template<typename _Tag>
  struct _Enable_copy_move<false, true, true, true, _Tag>
  {
    constexpr _Enable_copy_move() noexcept                          = default;
    constexpr _Enable_copy_move(_Enable_copy_move const&) noexcept  = delete;
    constexpr _Enable_copy_move(_Enable_copy_move&&) noexcept       = default;
    _Enable_copy_move&
    operator=(_Enable_copy_move const&) noexcept                    = default;
    _Enable_copy_move&
    operator=(_Enable_copy_move&&) noexcept                         = default;
  };
```

总的来看，`std::variant`的继承关系是C++“策略模式”和“元编程”的典型应用：通过多层基类的条件性实现，根据备选类型的特性（平凡性、可复制性等）动态调整行为，在保证类型安全和异常安全的同时，最大化性能。

### 其他帮助类型

通过前一章节，我们理解了 std::variant 的继承结构的组织方式。这一章节我们来认识一些起到辅助作用的元素

**1. `variant_size`：获取备选类型数量**

`variant_size` 利用 [`std::integral_constant`](https://en.cppreference.com/w/cpp/types/integral_constant.html) 和 [`sizeof... operator`](https://en.cppreference.com/w/cpp/language/sizeof....html) 实现计算 `variant` 模板参数个数的功能

```cpp
template <typename _Variant> struct variant_size;

// 特化：对于variant<_Types...>，返回备选类型数量
template <typename... _Types>
struct variant_size<variant<_Types...>>
    : std::integral_constant<size_t, sizeof...(_Types)> {};
```

#### 2. `variant_alternative`：获取指定索引的类型
```cpp
template <size_t _Np, typename _Variant> struct variant_alternative;

// 特化：对于variant<_Types...>，返回第_Np个类型
template <size_t _Np, typename... _Types>
struct variant_alternative<_Np, variant<_Types...>> {
  static_assert(_Np < sizeof...(_Types), "索引越界");
  using type = typename _Nth_type<_Np, _Types...>::type; // 元函数提取第N个类型
};

// 便捷别名
template <size_t _Np, typename _Variant>
using variant_alternative_t = typename variant_alternative<_Np, _Variant>::type;
```
- 功能：通过`variant_alternative_t<0, variant<int, double>>`可获取第0个类型（此处为`int`）。
- 依赖：`_Nth_type`是内部元函数，用于从类型包中提取第N个类型。


### 四、存储实现
`variant`的存储核心是**联合体（union）**，但需处理不同类型的构造/析构、对齐和异常安全，因此设计了多层封装。

#### 1. `_Uninitialized`：未初始化存储
```cpp
template <typename _Type, bool = std::is_trivially_destructible_v<_Type>>
struct _Uninitialized;

// 平凡析构类型：直接存储对象
template <typename _Type>
struct _Uninitialized<_Type, true> {
  _Type _M_storage; // 直接存储对象
  // 构造函数：原地构造
  template <typename... _Args>
  constexpr _Uninitialized(in_place_index_t<0>, _Args&&... __args)
      : _M_storage(std::forward<_Args>(__args)...) {}
};

// 非平凡析构类型：使用对齐缓冲区（C++17）
template <typename _Type>
struct _Uninitialized<_Type, false> {
  __gnu_cxx::__aligned_membuf<_Type> _M_storage; // 对齐的未初始化缓冲区
  // 构造函数： placement new 构造
  template <typename... _Args>
  constexpr _Uninitialized(in_place_index_t<0>, _Args&&... __args) {
    ::new ((void*)std::addressof(_M_storage)) _Type(std::forward<_Args>(__args)...);
  }
};
```
- 功能：根据类型是否平凡可析构，选择直接存储或对齐缓冲区存储，避免联合体中非平凡析构函数的问题（C++17中联合体默认不调用成员析构函数）。
- 优势：兼顾性能（平凡类型直接存储）和正确性（非平凡类型手动管理构造/析构）。


#### 2. `_Variadic_union`：递归联合体
```cpp
template <bool __trivially_destructible, typename _First, typename... _Rest>
union _Variadic_union {
  _Uninitialized<_First> _M_first; // 第一个类型的存储
  _Variadic_union<__trivially_destructible, _Rest...> _M_rest; // 剩余类型的递归存储

  // 构造函数：根据索引选择初始化第一个类型或剩余类型
  template <size_t _Np, typename... _Args>
  constexpr _Variadic_union(in_place_index_t<_Np>, _Args&&... __args)
      : _M_rest(in_place_index<_Np - 1>, std::forward<_Args>(__args)...) {}
};
```
- 功能：递归定义联合体，支持任意数量的备选类型（如`variant<int, double, string>`会生成包含`int`、`double`、`string`的嵌套联合体）。
- 内存布局：整体大小为所有备选类型的最大大小，对齐为最大对齐要求，避免内存浪费。


#### 3. `_Variant_storage`：管理活跃类型索引与析构
```cpp
template <bool __trivially_destructible, typename... _Types>
struct _Variant_storage {
  _Variadic_union<__trivially_destructible, _Types...> _M_u; // 存储联合体
  using __index_type = __select_index<_Types...>; // 最小化索引类型（如char、short）
  __index_type _M_index; // 活跃类型的索引（variant_npos表示无值）

  // 析构活跃类型（非平凡析构时）
  constexpr void _M_reset() {
    if (!_M_valid()) return;
    // 调用活跃类型的析构函数
    std::__do_visit([](auto&& __mem) { std::_Destroy(std::__addressof(__mem)); }, *this);
    _M_index = variant_npos;
  }

  // 检查是否有有效值（非valueless状态）
  constexpr bool _M_valid() const noexcept {
    return _M_index != variant_npos;
  }
};
```
- 核心成员：`_M_u`存储实际数据，`_M_index`记录当前活跃类型的索引（`variant_npos`表示因异常而无值）。
- 析构管理：`_M_reset`负责调用活跃类型的析构函数（仅非平凡析构类型需要），保证资源正确释放。


### 五、构造与赋值：异常安全设计
`variant`的构造和赋值需要处理类型转换、资源管理和异常安全，核心依赖于类型特性（`_Traits`）和条件实现。

#### 1. 类型特性`_Traits`
```cpp
template <typename... _Types> struct _Traits {
  static constexpr bool _S_copy_ctor = (is_copy_constructible_v<_Types> && ...);
  static constexpr bool _S_move_ctor = (is_move_constructible_v<_Types> && ...);
  static constexpr bool _S_trivial_dtor = (is_trivially_destructible_v<_Types> && ...);
  // 其他特性：复制/移动赋值的平凡性、异常安全性等
};
```
- 功能：封装备选类型的共性特性（如是否可复制、析构是否平凡），用于条件编译构造/赋值逻辑。


#### 2. 构造函数：支持直接初始化与类型转换
`variant`的构造函数分为：
- **直接初始化**：通过`in_place_index`或`in_place_type`指定类型，直接构造对象。
  ```cpp
  template <size_t _Np, typename... _Args>
  constexpr explicit variant(in_place_index_t<_Np>, _Args&&... __args)
      : _Base(in_place_index<_Np>, std::forward<_Args>(__args)...) {}
  ```
- **隐式转换**：从备选类型之一隐式转换（需唯一匹配）。
  ```cpp
  template <typename _Tp>
  constexpr variant(_Tp&& __t) // 仅当_Tp可唯一转换为某个备选类型时有效
      : variant(in_place_index<__accepted_index<_Tp>>, std::forward<_Tp>(__t)) {}
  ```


#### 3. 赋值操作：强异常安全保证
赋值操作需处理三种情况：
- 目标类型与当前类型相同：直接调用赋值运算符。
- 目标类型不同：销毁当前类型，构造新类型。
- 异常安全：若新类型构造可能抛出，通过临时`variant`中转（利用移动语义），保证原`variant`状态不变。
  ```cpp
  template <size_t _Np, typename... _Args>
  _GLIBCXX20_CONSTEXPR __to_type<_Np>& emplace(_Args&&... __args) {
    if constexpr (is_nothrow_constructible_v<type, _Args...>) {
      // 无异常风险：直接构造
      __variant::__emplace<_Np>(*this, std::forward<_Args>(__args)...);
    } else {
      // 有异常风险：通过临时对象中转
      variant __tmp(in_place_index<_Np>, std::forward<_Args>(__args)...);
      *this = std::move(__tmp); // 移动赋值（假设无异常）
    }
  }
  ```


### 六、访问机制：`get`与`visit`
`variant`的访问是类型安全的核心，禁止访问与活跃类型不符的值。

#### 1. `get`：直接按索引或类型访问
```cpp
// 按索引访问
template <size_t _Np, typename... _Types>
constexpr variant_alternative_t<_Np, variant<_Types...>>&
get(variant<_Types...>& __v) {
  if (__v.index() != _Np) // 检查索引是否匹配
    __throw_bad_variant_access(__v.valueless_by_exception()); // 不匹配则抛异常
  return __detail::__variant::__get<_Np>(__v); // 内部获取值
}

// 按类型访问（要求类型唯一）
template <typename _Tp, typename... _Types>
constexpr _Tp& get(variant<_Types...>& __v) {
  static_assert(__exactly_once<_Tp, _Types...>, "类型必须唯一");
  constexpr size_t __n = std::__find_uniq_type_in_pack<_Tp, _Types...>();
  return get<__n>(__v);
}
```
- 类型安全：访问前检查当前活跃类型的索引，不匹配则抛出`bad_variant_access`异常。
- 唯一性检查：按类型访问时，要求类型在备选列表中唯一（通过`__exactly_once`验证）。


#### 2. `visit`：多态访问（访问者模式）
`visit`是`variant`最强大的功能之一，允许用统一的访问器处理所有可能的活跃类型，实现类似多态的行为。

##### 实现原理：编译期生成跳转表
`visit`的核心是在编译期为所有可能的活跃类型组合生成函数指针表（跳转表），运行时根据当前索引直接调用对应函数。

```cpp
template <typename _Visitor, typename... _Variants>
constexpr auto visit(_Visitor&& __visitor, _Variants&&... __variants) {
  // 检查是否有valueless状态的variant
  if ((__variants.valueless_by_exception() || ...))
    __throw_bad_variant_access(2);

  // 生成跳转表：针对所有variant的活跃类型组合
  constexpr auto& __vtable = __gen_vtable<...>::_S_vtable;

  // 运行时根据索引调用对应函数
  auto __func_ptr = __vtable._M_access(__variants.index()...);
  return (*__func_ptr)(std::forward<_Visitor>(__visitor), ...);
}
```
- **跳转表生成**：`__gen_vtable`递归生成包含所有类型组合的函数指针表（如`variant<int, double>`和`variant<string>`的组合会生成4个函数指针）。
- **效率优化**：当备选类型数量较少时（≤11），使用`switch-case`替代跳转表，减少间接调用开销。


### 七、异常安全与`valueless_by_exception`
`variant`可能因构造新类型时抛出异常而进入`valueless_by_exception`状态（无有效数据），此时所有访问操作都会失败。

- **触发场景**：构造新类型时抛出异常（如`string`的构造失败），且无法回滚到原状态。
- **检测方式**：通过`valueless_by_exception()`函数判断，或通过`index() == variant_npos`判断。
- **恢复方式**：需显式重新赋值（如`v.emplace<0>(...)`）。


### 八、C++版本兼容性
源码通过宏（如`__cpp_lib_variant`）适配不同C++标准：
- **C++17**：基础功能，支持`visit`、`get`等。
- **C++20**：增强`constexpr`支持，添加三路比较运算符（`<=>`）。
- **C++26**：新增成员函数`visit`（`v.visit(visitor)`替代`std::visit(visitor, v)`）。


### 九、总结
`std::variant`的实现是C++元编程和异常安全设计的典范，核心亮点包括：
1. **类型安全**：通过编译期类型检查和运行时索引验证，避免传统`union`的未定义行为。
2. **高效存储**：递归联合体+对齐缓冲区，保证内存紧凑且正确对齐。
3. **异常安全**：通过临时对象中转和`valueless_by_exception`状态，提供可预测的错误处理。
4. **灵活访问**：`visit`函数通过编译期跳转表实现多态访问，兼顾灵活性与性能。

理解`variant`的源码有助于深入掌握C++类型系统、元编程技巧和异常安全设计原则。


2.2 类型安全保证
std::variant 提供了全面的类型安全保证，包括：
1. 编译期类型检查：
  - 只能存储模板参数中指定的类型
  - 模板参数必须是非数组的对象类型
  - 所有操作都有严格的类型约束
2. 运行时类型检查：
  - get<T>() 和 get<I>() 在类型不匹配时抛出 std::bad_variant_access 异常
  - get_if<T>() 和 get_if<I>() 在类型不匹配时返回 nullptr
  - holds_alternative<T>() 检查当前是否持有特定类型
3. 类型转换安全：
  - 赋值和构造时会根据重载解析规则选择最佳匹配类型
  - 禁止可能导致数据丢失的隐式转换
std::variant 的类型安全是在编译期和运行时两个层面共同保证的。编译期检查防止了无效类型的使用，而运行时检查确保了对当前持有类型的安全访问。
2.3 valueless_by_exception状态
std::variant 可能处于一个特殊的"无效"状态，称为"valueless by exception"。当在赋值或emplace操作期间发生异常，且无法保持原有状态时，variant将进入此状态。
特性与检测
std::variant<std::string, std::vector<int>> v = "hello";

// 检查是否处于无效状态
if (v.valueless_by_exception()) {
    std::cout << "variant处于无效状态" << std::endl;
}

// 无效状态下index()返回特殊值
if (v.index() == std::variant_npos) {
    std::cout << "variant处于无效状态" << std::endl;
}

// 访问无效状态会抛出异常
try {
    std::get<0>(v);  // 如果v无效，将抛出bad_variant_access
} catch (const std::bad_variant_access& e) {
    std::cerr << "访问错误: " << e.what() << std::endl;
}

产生无效状态的情况
struct ThrowOnCopy {
    ThrowOnCopy() = default;
    ThrowOnCopy(const ThrowOnCopy&) { throw std::runtime_error("Copy error"); }
};

std::variant<int, ThrowOnCopy> v = 10;

try {
    ThrowOnCopy t;
    v = t;  // 尝试赋值，将抛出异常
} catch (const std::exception& e) {
    std::cout << "捕获异常: " << e.what() << std::endl;
    
    // 此时v可能处于无效状态
    if (v.valueless_by_exception()) {
        std::cout << "variant现在处于无效状态" << std::endl;
    }
}

[图片]
3. 内部实现原理
3.1 内存布局与对齐机制
std::variant 的内存布局主要由两部分组成：
1. 存储区域：用于存储当前活动类型的值，大小至少为最大可能类型的大小
2. 索引：用于跟踪当前持有的类型，通常是一个整数
[图片]
std::variant 的总体大小约为：
sizeof(std::variant<Types...>) ≈ max(sizeof(Types...)) + sizeof(index_type)

其中，index_type 的大小根据类型数量自动选择：
- 如果类型数量 ≤ 256，使用 uint8_t
- 如果类型数量 ≤ 65536，使用 uint16_t
- 否则使用更大的整数类型
对齐要求遵循最严格的类型对齐要求，可能导致额外的填充字节：
alignof(std::variant<Types...>) = max(alignof(Types...))

在GCC的实现中，相关代码如下：
template <typename... _Types>
using __select_index =
  typename __select_int::_Select_int_base<sizeof...(_Types),
                                  unsigned char,
                                  unsigned short>::type::value_type;

3.2 类型索引跟踪
std::variant 使用一个索引成员来跟踪当前存储的类型：
using __index_type = __select_index<_Types...>;
__index_type _M_index;

索引值从0开始，对应于模板参数列表中的类型顺序。特殊值 variant_npos（通常是 size_t 的最大值）用于表示"valueless by exception"状态。
在GCC的实现中，相关代码如下：
inline constexpr size_t variant_npos = -1;

constexpr size_t index() const noexcept
{
  return this->_M_valid() ? this->_M_index : variant_npos;
}

constexpr bool _M_valid() const noexcept
{
  if constexpr (__variant::__never_valueless<_Types...>())
    return true;
  return this->_M_index != __index_type(variant_npos);
}

3.3 递归联合体实现
std::variant 的核心存储机制是一个递归联合体（recursive union），它允许存储任意数量的类型。在GCC的实现中，这个递归联合体称为 _Variadic_union：
template<bool __trivially_destructible, typename _First, typename... _Rest>
union _Variadic_union<__trivially_destructible, _First, _Rest...>
{
  _Uninitialized<_First> _M_first;
  _Variadic_union<__trivially_destructible, _Rest...> _M_rest;
  // ...
};

// 递归基础情况
template<bool __trivially_destructible, typename... _Types>
union _Variadic_union
{
  _Variadic_union() = default;
  
  template<size_t _Np, typename... _Args>
  _Variadic_union(in_place_index_t<_Np>, _Args&&...) = delete;
};

每个类型都被包装在 _Uninitialized 中，以处理不同的析构行为：
// 对于可平凡析构的类型
template<typename _Type, bool>
struct _Uninitialized
{
  template<typename... _Args>
  constexpr
  _Uninitialized(in_place_index_t<0>, _Args&&... __args)
  : _M_storage(std::forward<_Args>(__args)...)
  { }

  _Type _M_storage;
};

// 对于不可平凡析构的类型
template<typename _Type>
struct _Uninitialized<_Type, false>
{
  template<typename... _Args>
  constexpr
  _Uninitialized(in_place_index_t<0>, _Args&&... __args)
  {
    ::new ((void*)std::addressof(_M_storage))
      _Type(std::forward<_Args>(__args)...);
  }

  __gnu_cxx::__aligned_membuf<_Type> _M_storage;
};

这种设计使得 variant 可以存储任意数量的类型，同时正确处理构造和析构行为。
4. 类型安全实现机制
4.1 编译期类型检查
std::variant 使用多种编译期类型检查机制来确保类型安全：
[图片]
静态断言
静态断言用于在编译期验证类型满足基本要求：
static_assert(sizeof...(_Types) > 0,
        "variant must have at least one alternative");
        
static_assert(((std::is_object_v<_Types> && !is_array_v<_Types>) && ...),
        "variant alternatives must be non-array object types");

SFINAE约束
使用 enable_if_t 限制模板实例化，确保只有符合条件的类型才能使用特定操作：
template<typename _Tp, typename... _Args>
_GLIBCXX20_CONSTEXPR
enable_if_t<is_constructible_v<_Tp, _Args...> && __exactly_once<_Tp>,
            _Tp&>
emplace(_Args&&... __args);

类型特征检查
使用辅助模板确保类型唯一性和其他特性：
template<typename _Tp, typename... _Types>
inline constexpr bool __exactly_once
  = std::__find_uniq_type_in_pack<_Tp, _Types...>() < sizeof...(_Types);

4.2 运行时类型检查
在运行时，std::variant 使用索引来确保类型安全：
索引验证
在访问前验证索引与请求的类型匹配：
template<size_t _Np, typename... _Types>
constexpr variant_alternative_t<_Np, variant<_Types...>>&
get(variant<_Types...>& __v)
{
  static_assert(_Np < sizeof...(_Types),
        "The index must be in [0, number of alternatives)");
  if (__v.index() != _Np)
    __throw_bad_variant_access(__v.valueless_by_exception());
  return __detail::__variant::__get<_Np>(__v);
}

异常抛出
当访问错误类型或无效状态时抛出异常：
[[noreturn]] void __throw_bad_variant_access(unsigned);

4.3 类型转换安全
std::variant 的转换构造函数和赋值运算符使用了 __accepted_index 模板来确定目标类型：
template<typename _Tp>
static constexpr size_t __accepted_index
  = __detail::__variant::__accepted_index<_Tp, variant>;

__accepted_index 的实现使用了一个巧妙的技术，通过模拟函数重载解析来确定最佳匹配类型：
template<typename _Tp, typename _Variant>
using _FUN_type
  = decltype(_Build_FUNs<_Tp, _Variant>::_S_fun(std::declval<_Tp>()));

template<typename _Tp, typename _Variant, typename = void>
inline constexpr size_t
__accepted_index = variant_npos;

template<typename _Tp, typename _Variant>
inline constexpr size_t
__accepted_index<_Tp, _Variant, void_t<_FUN_type<_Tp, _Variant>>>
  = _FUN_type<_Tp, _Variant>::value;

这种设计确保了类型转换遵循C++标准中定义的规则，避免了意外的隐式转换。
5. 访问者模式设计与实现
5.1 std::visit的工作原理
std::visit 是 std::variant 提供的一种强大的访问机制，它实现了访问者模式（Visitor Pattern）：
template<typename _Visitor, typename... _Variants>
constexpr auto visit(_Visitor&& __visitor, _Variants&&... __variants);

[图片]
std::visit 的基本工作原理是：
1. 检查所有variant参数是否有任何一个处于无效状态，如有则抛出异常
2. 确定返回类型（通常是访问者函数对所有可能类型组合的返回类型）
3. 根据variant的当前索引，调用访问者函数处理对应的类型
template<typename _Visitor, typename... _Variants>
constexpr __detail::__variant::__visit_result_t<_Visitor, _Variants...>
visit(_Visitor&& __visitor, _Variants&&... __variants)
{
  namespace __variant = std::__detail::__variant;

  if ((__variant::__as(__variants).valueless_by_exception() || ...))
    __throw_bad_variant_access(2);

  using _Result_type
    = __detail::__variant::__visit_result_t<_Visitor, _Variants...>;

  using _Tag = __detail::__variant::__deduce_visit_result<_Result_type>;

  // ...
  
  return std::__do_visit<_Tag>(
    std::forward<_Visitor>(__visitor),
    __variant::__as(std::forward<_Variants>(__variants))...);
}

5.2 多维虚表生成
std::visit 的核心实现依赖于编译期生成的多维虚表（multi-dimensional virtual table）：
template<typename _Result_type, typename _Visitor, typename... _Variants>
struct __gen_vtable
{
  using _Array_type =
    _Multi_array<_Result_type (*)(_Visitor, _Variants...),
                 variant_size_v<remove_reference_t<_Variants>>...>;

  static constexpr _Array_type _S_vtable
    = __gen_vtable_impl<_Array_type, std::index_sequence<>>::_S_apply();
};

_Multi_array 是一个多维数组，其维度由variant类型数量和每个variant的替代项数量决定：
template<typename _Tp, size_t... _Dimensions>
struct _Multi_array;

template<typename _Tp>
struct _Multi_array<_Tp>
{
  // ...
  typename __untag_result<_Tp>::element_type _M_data;
};

template<typename _Ret, typename _Visitor, typename... _Variants,
         size_t __first, size_t... __rest>
struct _Multi_array<_Ret(*)(_Visitor, _Variants...), __first, __rest...>
{
  // ...
  _Multi_array<_Tp, __rest...> _M_arr[__first + __do_cookie];
};

虚表通过递归模板实例化在编译期填充，为每种可能的类型组合生成一个函数指针。
5.3 访问者模式优化策略
为了提高性能，std::visit 实现了多种优化策略：
单一variant优化
对于单个variant和少量替代项的情况，使用switch-case优化而非虚表：
if constexpr (sizeof...(_Variants) > 1 || __n > __max)
{
  // 使用虚表
  constexpr auto& __vtable = __detail::__variant::__gen_vtable<
    _Result_type, _Visitor&&, _Variants&&...>::_S_vtable;

  auto __func_ptr = __vtable._M_access(__variants.index()...);
  return (*__func_ptr)(std::forward<_Visitor>(__visitor),
                       std::forward<_Variants>(__variants)...);
}
else // 单个variant，少量替代项
{
  // 使用switch-case优化
  switch (__v0.index())
  {
    _GLIBCXX_VISIT_CASE(0)
    _GLIBCXX_VISIT_CASE(1)
    // ...
    _GLIBCXX_VISIT_CASE(10)
    // ...
  }
}

访问者示例
// 定义访问者
struct Visitor {
    void operator()(int i) { std::cout << "整数: " << i << std::endl; }
    void operator()(const std::string& s) { std::cout << "字符串: " << s << std::endl; }
    void operator()(double d) { std::cout << "浮点数: " << d << std::endl; }
};

// 创建variant
std::variant<int, std::string, double> v = 42;

// 使用访问者
std::visit(Visitor{}, v);  // 输出: 整数: 42

// 使用泛型lambda作为访问者
std::visit([](const auto& val) {
    using T = std::decay_t<decltype(val)>;
    if constexpr (std::is_same_v<T, int>)
        std::cout << "整数: " << val << std::endl;
    else if constexpr (std::is_same_v<T, std::string>)
        std::cout << "字符串: " << val << std::endl;
    else
        std::cout << "浮点数: " << val << std::endl;
}, v);

[图片]
6. valueless_by_exception处理机制
6.1 无效状态的产生与检测
std::variant 可能在某些情况下变为无效状态，例如在异常导致的部分构造后。这种状态称为"valueless by exception"。
[图片]
无效状态的产生
无效状态主要在以下情况下产生：
1. 当尝试赋值或替换当前值时发生异常
2. 当无法保持原有状态且无法完成新状态的构造时
struct MayThrow {
    MayThrow() = default;
    MayThrow(const MayThrow&) { 
        if (std::rand() % 2) throw std::runtime_error("随机构造失败");
    }
};

std::variant<int, MayThrow> v = 10;

try {
    MayThrow m;
    v = m;  // 可能抛出异常
} catch (...) {
    // 此时v可能处于无效状态
    if (v.valueless_by_exception())
        std::cout << "v现在无效" << std::endl;
}

无效状态的检测
std::variant 提供了 valueless_by_exception() 方法来检测无效状态：
constexpr bool valueless_by_exception() const noexcept
{ return !this->_M_valid(); }

constexpr bool
_M_valid() const noexcept
{
  if constexpr (__variant::__never_valueless<_Types...>())
    return true;
  return this->_M_index != __index_type(variant_npos);
}

此外，index() 方法在无效状态下返回特殊值 variant_npos。
6.2 Never Valueless优化
GCC实现了一个重要优化，称为"Never Valueless"优化。对于某些类型组合，可以保证variant永远不会处于无效状态：
template<typename _Tp>
struct _Never_valueless_alt
: __and_<bool_constant<sizeof(_Tp) <= 256>, is_trivially_copyable<_Tp>>
{ };

template <typename... _Types>
constexpr bool __never_valueless()
{
  return _Traits<_Types...>::_S_move_assign
    && (_Never_valueless_alt<_Types>::value && ...);
}

如果所有类型都满足以下条件，则variant永远不会处于无效状态：
1. 大小不超过256字节
2. 可平凡复制
3. variant支持移动赋值
这种优化可以简化实现并提高性能，因为不需要处理无效状态的特殊情况。
6.3 异常安全保证
std::variant 的 emplace 方法实现了不同级别的异常安全保证：
template<size_t _Np, typename... _Args>
_GLIBCXX20_CONSTEXPR
enable_if_t<is_constructible_v<__to_type<_Np>, _Args...>,
            __to_type<_Np>&>
emplace(_Args&&... __args)
{
  // ...
  // 强异常安全保证
  if constexpr (is_nothrow_constructible_v<type, _Args...>)
  {
    __variant::__emplace<_Np>(*this, std::forward<_Args>(__args)...);
  }
  else if constexpr (is_scalar_v<type>)
  {
    // 标量类型的特殊处理
    const type __tmp(std::forward<_Args>(__args)...);
    __variant::__emplace<_Np>(*this, __tmp);
  }
  else if constexpr (__variant::_Never_valueless_alt<type>()
      && _Traits::_S_move_assign)
  {
    // 使用临时variant
    variant __tmp(in_place_index<_Np>,
                std::forward<_Args>(__args)...);
    *this = std::move(__tmp);
  }
  else
  {
    // 基本异常安全保证
    __variant::__emplace<_Np>(*this, std::forward<_Args>(__args)...);
  }
  // ...
}

根据类型特性，emplace 方法会选择不同的实现策略：
1. 对于 nothrow 构造函数，直接在原位构造（强异常安全保证）
2. 对于标量类型，先构造临时对象再赋值（强异常安全保证）
3. 对于满足"Never Valueless"条件的类型，使用临时variant（强异常安全保证）
4. 对于其他类型，提供基本异常安全保证
7. 性能优化策略
7.1 内存布局优化
std::variant 实现了多种内存布局优化：
索引类型优化
根据类型数量选择最小的整数类型来存储索引：
template <typename... _Types>
using __select_index =
  typename __select_int::_Select_int_base<sizeof...(_Types),
                                  unsigned char,
                                  unsigned short>::type::value_type;

这种优化可以减少variant的内存占用，特别是对于少量类型的情况。
对齐优化
std::variant 使用 __aligned_membuf 来确保正确的内存对齐，同时避免不必要的填充：
template<typename _Type>
struct _Uninitialized<_Type, false>
{
  // ...
  __gnu_cxx::__aligned_membuf<_Type> _M_storage;
};

7.2 特殊成员函数优化
GCC使用 _Traits 类和多层继承来有条件地启用或禁用特殊成员函数：
template<typename... _Types>
struct _Traits
{
  static constexpr bool _S_default_ctor =
    is_default_constructible_v<typename _Nth_type<0, _Types...>::type>;
  static constexpr bool _S_copy_ctor =
    (is_copy_constructible_v<_Types> && ...);
  static constexpr bool _S_move_ctor =
    (is_move_constructible_v<_Types> && ...);
  // ...
};

然后使用 _Enable_copy_move 基类来控制特殊成员函数的生成：
template<typename... _Types>
class variant
  : private __detail::__variant::_Variant_base<_Types...>,
    private _Enable_copy_move<
      __detail::__variant::_Traits<_Types...>::_S_copy_ctor,
      __detail::__variant::_Traits<_Types...>::_S_copy_assign,
      __detail::__variant::_Traits<_Types...>::_S_move_ctor,
      __detail::__variant::_Traits<_Types...>::_S_move_assign,
      variant<_Types...>>
{
  // ...
};

这种设计确保了特殊成员函数只在所有类型都支持相应操作时才启用，避免了不必要的代码生成。
7.3 访问性能优化
std::variant 实现了多种访问性能优化：
访问器函数优化
get 和 get_if 函数的实现使用了递归模板来高效访问variant中的值：
template<size_t _Np, typename _Union>
constexpr decltype(auto)
__get_n(_Union&& __u) noexcept
{
  if constexpr (_Np == 0)
    return std::forward<_Union>(__u)._M_first._M_get();
  else if constexpr (_Np == 1)
    return std::forward<_Union>(__u)._M_rest._M_first._M_get();
  else if constexpr (_Np == 2)
    return std::forward<_Union>(__u)._M_rest._M_rest._M_first._M_get();
  else
    return __variant::__get_n<_Np - 3>(
             std::forward<_Union>(__u)._M_rest._M_rest._M_rest);
}

对于常用的前几个索引，使用直接访问而不是递归，这可以提高性能。
访问者模式优化
如前所述，std::visit 对单个variant和少量替代项使用switch-case优化，避免了虚表的开销。
[图片]
8. 与其他类似概念的比较
8.1 与传统union的比较
[图片]
std::variant 相比传统的C风格联合体有以下优势：
1. 类型安全：自动跟踪当前类型，提供类型检查
2. 支持非POD类型：可以存储具有构造函数、析构函数的类型
3. 自动析构：正确调用当前持有对象的析构函数
4. 异常安全：提供异常安全保证
但也有一些劣势：
1. 内存开销：需要额外存储类型索引
2. 性能开销：类型检查和异常处理带来轻微性能损失
3. 实现复杂度：实现更复杂，可能导致代码膨胀
8.2 与boost::variant的比较
[图片]
std::variant 与 Boost.Variant 的主要区别：
1. 无值状态：std::variant 支持valueless_by_exception状态，而Boost.Variant保证never-empty
2. 递归变体：std::variant 不直接支持递归定义，而Boost.Variant通过recursive_wrapper支持
3. 访问API：std::variant 使用std::visit和std::get，而Boost.Variant使用apply_visitor和get
4. 引用支持：std::variant 不支持引用类型，而Boost.Variant可通过wrapper支持
5. 性能：std::variant 通常更优化，但功能相对较少
8.3 与std::any的比较
[图片]
std::variant 与 std::any 的主要区别：
1. 类型安全：std::variant 限制为预定义类型集，而 std::any 可存储任意类型
2. 编译期检查：std::variant 提供编译期类型检查，而 std::any 只有运行时检查
3. 内存模型：std::variant 通常在栈上分配，而 std::any 可能需要堆分配
4. 性能：std::variant 通常更高效，没有类型擦除开销
5. 使用场景：std::variant 适用于已知类型集，std::any 适用于完全动态类型
8.4 与基于继承的多态比较
[图片]
std::variant 与基于继承的多态比较：
1. 类型关系：std::variant 不需要类型之间有继承关系，而多态要求共同基类
2. 内存模型：std::variant 使用值语义，通常在栈上，而多态通常使用引用语义，在堆上
3. 扩展性：std::variant 类型集在编译期固定，而多态可在运行时扩展
4. 性能：std::variant 通常更快，缓存友好，没有虚函数调用开销
5. 代码复杂度：std::variant 使用模板元编程，而多态使用更熟悉的面向对象设计
9. 实际应用场景和最佳实践
9.1 典型应用场景
std::variant 在以下场景特别有用：
状态机实现
// 定义状态
struct Idle {};
struct Running { int progress; };
struct Failed { std::string error; };

// 状态机
class Task {
    std::variant<Idle, Running, Failed> state_;
public:
    Task() : state_(Idle{}) {}
    
    void start() {
        state_ = Running{0};
    }
    
    void update() {
        std::visit(overloaded {
            [this](Idle&) { /* 不做任何事 */ },
            [this](Running& r) {
                r.progress += 10;
                if (r.progress >= 100)
                    state_ = Idle{};
            },
            [](Failed&) { /* 处理错误 */ }
        }, state_);
    }
};

异构容器
// 可以存储不同类型的集合
std::vector<std::variant<int, std::string, double>> collection;
collection.push_back(42);
collection.push_back("Hello");
collection.push_back(3.14);

// 处理所有元素
for (const auto& item : collection) {
    std::visit([](const auto& val) { std::cout << val << std::endl; }, item);
}

函数返回多种类型
// 函数可以返回不同类型
std::variant<std::string, std::vector<int>, std::error_code> 
parse_input(const std::string& input) {
    if (input.empty())
        return std::error_code(EINVAL, std::generic_category());
    
    if (input[0] == '[')
        return std::vector<int>{1, 2, 3}; // 解析为数组
    
    return input; // 返回原始字符串
}

9.2 使用模式与技巧
overloaded辅助类
定义一个辅助类简化访问者定义：
// overloaded辅助类
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>; // C++17指引推导

// 使用示例
std::variant<int, std::string> v = "hello";
std::visit(overloaded {
    [](int i) { std::cout << "整数: " << i << std::endl; },
    [](const std::string& s) { std::cout << "字符串: " << s << std::endl; }
}, v);

返回值访问者
使用访问者返回统一类型：
auto result = std::visit([](const auto& val) -> std::string {
    using std::to_string;
    if constexpr (std::is_same_v<std::decay_t<decltype(val)>, std::string>)
        return val;
    else
        return to_string(val);
}, v);

多个variant组合访问
std::variant<int, float> v1 = 42;
std::variant<std::string, bool> v2 = "hello";

std::visit([](auto&& a, auto&& b) {
    std::cout << a << " - " << b << std::endl;
}, v1, v2);

9.3 性能考量与建议
类型选择建议
- 限制类型数量：过多类型会增加编译时间和二进制大小
- 考虑类型大小：variant大小由最大类型决定，避免不必要的大型类型
- 避免重复类型：不要在同一个variant中包含相同类型
- 考虑默认构造顺序：将最可能使用的类型放在第一位
异常安全考量
- 注意valueless_by_exception状态：在修改variant时处理可能的异常
- 使用get_if而非get：优先使用返回指针的get_if以避免异常
- 考虑nothrow保证：如果所有类型都有nothrow移动构造，variant不会进入valueless状态
编译时影响
- 模板实例化：每个variant类型组合都会生成新代码
- 访问者代码膨胀：特别是多个variant的组合访问
- 内联扩展：许多操作在编译时展开，增加代码大小
优化建议：
- 限制variant中的类型数量，通常不超过10种
- 将复杂variant定义移至实现文件
- 考虑使用显式模板实例化控制代码膨胀
- 测量variant在性能关键代码中的实际影响
10. 参考资料与进一步阅读
标准文档
- ISO/IEC 14882:2017 - C++17标准，第23.7节
- P0088R3 - Variant: a type-safe union for C++17
实现源码
- GCC libstdc++ variant实现
- LLVM libc++ variant实现
- MSVC STL variant实现
相关文章与演讲
- Axel Naumann, "Variants: Past, Present, and Future" (CppCon 2015)
- Miro Knejp, "Implementing variant Visitation Using Lambdas" (CppCon 2018)
- Simon Brand, "How to Use C++17 std::variant" (2018)
- Arthur O'Dwyer, "The Best Type Traits C++ Doesn't Have" (CppCon 2021)
相关库
- Boost.Variant
- Boost.Variant2
- mpark/variant - C++14兼容的variant实现