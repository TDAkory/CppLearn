# 详解[`std::variant`](https://en.cppreference.com/w/cpp/utility/variant.html)

- [详解`std::variant`](#详解stdvariant)
  - [基本概念](#基本概念)
  - [源码分析](#源码分析)
    - [继承体系](#继承体系)
      - [`_Variant_base`继承链](#_variant_base继承链)
      - [`_Enable_copy_move`控制成员函数的生成](#_enable_copy_move控制成员函数的生成)
    - [其他帮助类型](#其他帮助类型)
      - [1. `std::monostate`](#1-stdmonostate)
      - [2. `variant_size`：获取备选类型数量](#2-variant_size获取备选类型数量)
      - [3. `variant_alternative`：获取指定索引的类型](#3-variant_alternative获取指定索引的类型)
      - [4. `hash<variant<typename... _Types>>`](#4-hashvarianttypename-_types)
      - [5. `valueless_by_exception`](#5-valueless_by_exception)
      - [6. `__never_valueless`优化](#6-__never_valueless优化)
    - [常用接口](#常用接口)
      - [构造函数：支持直接初始化与类型转换](#构造函数支持直接初始化与类型转换)
      - [赋值运算符](#赋值运算符)
      - [`emplace`操作](#emplace操作)
      - [`get`通过索引或类型访问](#get通过索引或类型访问)
      - [`visit`](#visit)
        - [例子](#例子)
  - [一些引申的思考](#一些引申的思考)
    - [为什么`struct _Uninitialized`需要针对`std::is_trivially_destructible_v`进行特化](#为什么struct-_uninitialized需要针对stdis_trivially_destructible_v进行特化)
    - [内存分析](#内存分析)
    - [与其他工具的对比](#与其他工具的对比)
  - [用法示例](#用法示例)
    - [构造](#构造)
    - [访问](#访问)
    - [修改](#修改)
    - [通过`overload`辅助类实现类switch-case语义](#通过overload辅助类实现类switch-case语义)
    - [结合`overload`实现状态机](#结合overload实现状态机)
    - [返回统一类型](#返回统一类型)
    - [多个`variant`组合访问](#多个variant组合访问)
    - [通过`variant`返回不同类型的值](#通过variant返回不同类型的值)
    - [利用`variant`实现异构容器](#利用variant实现异构容器)

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

## 源码分析

> gcc           libstdc++-v3/include/std/variant
>
> llvm-libcxx   libcxx/include/variant

本文主要分析GCC `releases/gcc-15.2.0`的实现（位于`libstdc++-v3/include/std/variant`）

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
// 内存布局：整体大小为所有备选类型的最大大小，对齐为最大对齐要求，避免内存浪费
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
  constexpr void _M_reset() {
    if (!_M_valid()) return;
    // 调用活跃类型的析构函数
    std::__do_visit([](auto&& __mem) { std::_Destroy(std::__addressof(__mem)); }, *this);
    _M_index = variant_npos;
  }

  // 检查是否有有效值（非valueless状态）
  constexpr bool _M_valid() const noexcept { return _M_index != variant_npos; }
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

#### 1. `std::monostate`

作为 `std::variant` 的“空状态”或“占位符”类型，用于解决 `std::variant` 无默认构造函数的场景（当所有备选类型都不可默认构造时，加入 `std::monostate` 可使 `std::variant` 支持默认构造）。

- 无状态（不包含任何数据成员）。
- 支持所有基础操作：默认构造、复制/移动构造、复制/移动赋值、析构（均为 `noexcept`）。
- 支持比较操作（`==`、`!=`、`<=>` 等），所有实例均相等。

```cpp
// libstdc++-v3/include/bits/monostate.h
  struct monostate { };

  constexpr bool operator==(monostate, monostate) noexcept { return true; }
#ifdef __cpp_lib_three_way_comparison
  constexpr strong_ordering
  operator<=>(monostate, monostate) noexcept { return strong_ordering::equal; }
#else
  constexpr bool operator!=(monostate, monostate) noexcept { return false; }
  constexpr bool operator<(monostate, monostate) noexcept { return false; }
  constexpr bool operator>(monostate, monostate) noexcept { return false; }
  constexpr bool operator<=(monostate, monostate) noexcept { return true; }
  constexpr bool operator>=(monostate, monostate) noexcept { return true; }
#endif
```

#### 2. `variant_size`：获取备选类型数量

`variant_size` 利用 [`std::integral_constant`](https://en.cppreference.com/w/cpp/types/integral_constant.html) 和 [`sizeof... operator`](https://en.cppreference.com/w/cpp/language/sizeof....html) 实现计算 `variant` 模板参数个数的功能

```cpp
template <typename _Variant> struct variant_size;

// 特化：对于variant<_Types...>，返回备选类型数量
template <typename... _Types>
struct variant_size<variant<_Types...>>
    : std::integral_constant<size_t, sizeof...(_Types)> {};
```

在 C++ 标准库中，`std::variant` 作为联合体（union）的类型安全封装，依赖多个辅助模板类和特性（traits）来实现其功能。结合提供的源码片段，以下是与 `std::variant` 相关的核心帮助模板类的详细解释：

#### 3. `variant_alternative`：获取指定索引的类型

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

```cpp
// libstdc++-v3/include/bits/utility.h
#if _GLIBCXX_USE_BUILTIN_TRAIT(__type_pack_element)
  template<size_t _Np, typename... _Types>
    struct _Nth_type
    { using type = __type_pack_element<_Np, _Types...>; };
#else
  template<size_t _Np, typename... _Types>
    struct _Nth_type
    { };

  template<typename _Tp0, typename... _Rest>
    struct _Nth_type<0, _Tp0, _Rest...>
    { using type = _Tp0; };

  template<typename _Tp0, typename _Tp1, typename... _Rest>
    struct _Nth_type<1, _Tp0, _Tp1, _Rest...>
    { using type = _Tp1; };

  template<typename _Tp0, typename _Tp1, typename _Tp2, typename... _Rest>
    struct _Nth_type<2, _Tp0, _Tp1, _Tp2, _Rest...>
    { using type = _Tp2; };
#endif
```

#### 4. `hash<variant<typename... _Types>>`

```cpp
/// @cond undocumented
template <typename... _Types> struct __variant_hash {
#if __cplusplus < 202002L
  using result_type [[__deprecated__]] = size_t;
  using argument_type [[__deprecated__]] = variant<_Types...>;
#endif
  // 使用折叠表达式检查所有类型的哈希操作是否都是 noexcept
  size_t operator()(const variant<_Types...> &__t) const
      noexcept((is_nothrow_invocable_v<hash<decay_t<_Types>>, _Types> && ...)) {  
    size_t __ret;
    __detail::__variant::__raw_visit(
        [&__t, &__ret](auto &&__t_mem) mutable {
          using _Type = __remove_cvref_t<decltype(__t_mem)>;
          if constexpr (!is_same_v<_Type,
                                   __detail::__variant::__variant_cookie>)
            __ret =
                std::hash<size_t>{}(__t.index()) + std::hash<_Type>{}(__t_mem);
          else
            __ret = std::hash<size_t>{}(__t.index());
        },
        __t);
    return __ret;
  }
};
/// @endcond

template <typename... _Types>
struct hash<variant<_Types...>>
    : __conditional_t<(__is_hash_enabled_for<remove_const_t<_Types>> && ...),
                      __variant_hash<_Types...>,
                      __hash_not_enabled<variant<_Types...>>>
{
};
```

- 使用折叠表达式检查所有类型的哈希操作是否都是 noexcept: `(is_nothrow_invocable_v<hash<decay_t<_Types>>, _Types> && ...)` 展开为：
  `is_nothrow_invocable_v<hash<T1>, T1> && is_nothrow_invocable_v<hash<T2>, T2> && ...`
- 使用 `__raw_visit` 直接访问 variant 的当前存储值
  - 如果当前存储的不是 `__variant_cookie`（表示有效值），计算组合哈希
  - 如果是 `__variant_cookie`（无效状态），只使用索引哈希
- 使用 `__conditional_t` 在编译时选择基类
  - **条件**：`(__is_hash_enabled_for<remove_const_t<_Types>> && ...)`
    - 检查所有 `_Types` 是否都支持哈希
    - 使用折叠表达式确保所有类型都有可用的 `std::hash` 特化
  - 如果所有类型都可哈希：继承 `__variant_hash<_Types...>`
  - 如果有类型不可哈希：继承 `__hash_not_enabled<variant<_Types...>>`（这通常会导致编译时错误，提供清晰的错误信息）

#### 5. `valueless_by_exception`

`variant`可能因构造新类型时抛出异常而进入`valueless_by_exception`状态（无有效数据），此时所有访问操作都会失败。

- **触发场景**：构造新类型时抛出异常（如`string`的构造失败），且无法回滚到原状态。
- **检测方式**：通过`valueless_by_exception()`函数判断，或通过`index() == variant_npos`判断。
- **恢复方式**：需显式重新赋值（如`v.emplace<0>(...)`）。

```cpp
// godbolt: https://godbolt.org/z/6hczvPc1r
#include <variant>
#include <iostream>
#include <assert.h>

struct ThrowOnCopy {
    ThrowOnCopy() = default;
    ThrowOnCopy(const ThrowOnCopy&) { throw std::runtime_error("Copy error"); }
};

int main() {
    std::variant<int, ThrowOnCopy> v = 10;

    try {
        ThrowOnCopy t;
        v = t;  // 尝试赋值，将抛出异常
    } catch (const std::exception& e) {
        std::cout << "捕获异常: " << e.what() << std::endl;
    
        // 此时v可能处于无效状态
        if (v.valueless_by_exception()) {
            assert(v.index() == std::variant_npos);
            std::cout << "variant现在处于无效状态" << std::endl;
        }
    }
    return 0;
}

// 捕获异常: Copy error
// variant现在处于无效状态
```

本质上，`valueless_by_exception` 是 `variant` 的成员函数，它是异常安全的，可以在编译期求值，通过 `_M_valid` 接口，最终根据 `variant._M_index` 来判断是否有值

```cpp
  constexpr bool valueless_by_exception() const noexcept {
    return !this->_M_valid();
  }

  // _M_valid() 方法实现，在 _Variant_storage 中有两个版本的实现

  // _Variant_storage<false, _Types...>，不可平凡析构的版本
  constexpr bool _M_valid() const noexcept {
    if constexpr (__variant::__never_valueless<_Types...>())
        return true;
    return this->_M_index != __index_type(variant_npos);
  }

  // _Variant_storage<true, _Types...>，可平凡析构的版本
  constexpr bool _M_valid() const noexcept {
    if constexpr (__variant::__never_valueless<_Types...>())
        return true;
    return this->_M_index != static_cast<__index_type>(variant_npos);
  }
```

#### 6. `__never_valueless`优化

GCC实现了一个重要优化，称为"Never Valueless"优化。对于某些类型组合，可以保证variant永远不会处于无效状态：

```cpp
template <typename _Tp>
struct _Never_valueless_alt   // 类型大小 ≤ 256 字节，且，类型是可平凡复制的
    : __and_<bool_constant<sizeof(_Tp) <= 256>, is_trivially_copyable<_Tp>> {};

// True if every alternative in _Types... can be emplaced in a variant
// without it becoming valueless. If this is true, variant<_Types...>
// can never be valueless, which enables some minor optimizations.
template <typename... _Types> constexpr bool __never_valueless() {
  return _Traits<_Types...>::_S_move_assign &&            // 所有类型都支持移动赋值
         (_Never_valueless_alt<_Types>::value && ...);    // 每个类型都满足"永远不会无值"的条件
}
```

如果所有类型都满足以下条件，则variant永远不会处于无效状态：

1. 大小不超过256字节
2. 可平凡复制
3. variant支持移动赋值

对于小型的、可平凡复制的类型，可以在栈上创建临时对象然后内存拷贝，避免因异常导致无值状态。

### 常用接口

#### 构造函数：支持直接初始化与类型转换

`variant`的构造函数分为：

1. **直接初始化**：通过`in_place_index`或`in_place_type`指定类型，直接构造对象。

```cpp
template <size_t _Np, typename... _Args>
constexpr explicit variant(in_place_index_t<_Np>, _Args&&... __args)
    : _Base(in_place_index<_Np>, std::forward<_Args>(__args)...) {}
```

2. **隐式转换**：从备选类型之一隐式转换（需唯一匹配）。

```cpp
template <typename _Tp>
constexpr variant(_Tp&& __t) // 仅当_Tp可唯一转换为某个备选类型时有效
    : variant(in_place_index<__accepted_index<_Tp>>, std::forward<_Tp>(__t)) {}
```

#### 赋值运算符

```cpp
  template <typename _Tp>
  _GLIBCXX20_CONSTEXPR
      enable_if_t<__exactly_once<__accepted_type<_Tp &&>> &&
                      is_constructible_v<__accepted_type<_Tp &&>, _Tp> &&
                      is_assignable_v<__accepted_type<_Tp &&> &, _Tp>,
                  variant &>
      operator=(_Tp &&__rhs) noexcept(  // 只有赋值和构造都是 noexcept 时，整个操作才是 noexcept
          is_nothrow_assignable_v<__accepted_type<_Tp &&> &, _Tp> &&
          is_nothrow_constructible_v<__accepted_type<_Tp &&>, _Tp>) {
    constexpr auto __index = __accepted_index<_Tp>; // 在编译时确定 `_Tp` 对应的 variant 成员索引
    if (index() == __index)   // 如果 variant 当前持有的就是目标类型，直接调用该类型的赋值运算符
      std::get<__index>(*this) = std::forward<_Tp>(__rhs);  // 使用 `std::forward<_Tp>` 保持值类别
    else {
      using _Tj = __accepted_type<_Tp &&>;
      // 这里使用 编译时分支 优化异常安全
      // 目标类型可以从 `_Tp` 无异常构造，或者 目标类型不是无异常移动构造的
      if constexpr (is_nothrow_constructible_v<_Tj, _Tp> ||
                    !is_nothrow_move_constructible_v<_Tj>)
        this->emplace<__index>(std::forward<_Tp>(__rhs));   // 直接构造（更高效）
      else
        // _GLIBCXX_RESOLVE_LIB_DEFECTS
        // 3585. converting assignment with immovable alternative
        // 如果 `emplace` 中直接构造可能抛出异常，而目标类型可以无异常移动
        // 先构造临时对象，然后无异常移动到最终位置
        // 提供强异常安全保证
        this->emplace<__index>(_Tj(std::forward<_Tp>(__rhs)));  // 先构造临时对象（更安全）
        
    }
    return *this;
  }
```

这个模板的约束条件解析：

1. **`__exactly_once<__accepted_type<_Tp &&>>`**
   - `__accepted_type<_Tp &&>`：找到能接受 `_Tp&&` 的 variant 成员类型
   - `__exactly_once`：确保该类型在 variant 的类型列表中**只出现一次**
   - 防止赋值时的二义性

2. **`is_constructible_v<__accepted_type<_Tp &&>, _Tp>`**
   - 检查是否可以从 `_Tp` 构造目标类型

3. **`is_assignable_v<__accepted_type<_Tp &&> &, _Tp>`**
   - 检查是否可以将 `_Tp` 赋值给目标类型

#### `emplace`操作

```cpp
  // 只有当可以从 `_Args...` 构造目标类型时才启用此重载
  template <size_t _Np, typename... _Args>
  _GLIBCXX20_CONSTEXPR enable_if_t<is_constructible_v<__to_type<_Np>, _Args...>,
                                   __to_type<_Np> &>
  emplace(_Args &&...__args) {
    namespace __variant = std::__detail::__variant;
    using type = typename _Nth_type<_Np, _Types...>::type;
    // Provide the strong exception-safety guarantee when possible,
    // to avoid becoming valueless.
    if constexpr (is_nothrow_constructible_v<type, _Args...>) {   // 无异常直接构造（最优）目标类型可以从参数无异常构造
      // 直接在 variant 存储中构造对象,强异常安全保证（不会使 variant 变为无值状态）
      __variant::__emplace<_Np>(*this, std::forward<_Args>(__args)...); 
    } else if constexpr (is_scalar_v<type>) {                     // 标量类型的优化处理
      // This might invoke a potentially-throwing conversion operator:
      const type __tmp(std::forward<_Args>(__args)...);     // 先构造临时标量对象（可能抛出异常）
      // But this won't throw:        
      __variant::__emplace<_Np>(*this, __tmp);              // 将临时对象复制到 variant 中（不会抛出）
    } else if constexpr (__variant::_Never_valueless_alt<type>() &&   
                         _Traits::_S_move_assign) {         
      // 强异常安全保证的构造,目标类型满足 `_Never_valueless_alt`（某些类型永远不会使 variant 无值）
      // This construction might throw:
      variant __tmp(in_place_index<_Np>, std::forward<_Args>(__args)...);
      // But _Never_valueless_alt<type> means this won't:
      *this = std::move(__tmp);   // 如果构造失败，当前 variant 保持原状
    } else {                      // 基本异常安全保证（最后手段）
      // This case only provides the basic exception-safety guarantee,
      // i.e. the variant can become valueless.
      // 直接在 variant 存储中构造，可能抛出异常,（variant 可能变为无值状态）
      __variant::__emplace<_Np>(*this, std::forward<_Args>(__args)...);
    }
    return std::get<_Np>(*this);
  }
```

这个 `emplace` 实现体现了 C++ 标准库对异常安全的高度重视：

- **编译时优化**：通过 `if constexpr` 选择最优策略
- **分层异常安全**：提供从强到基本的多种保证级别
- **零开销抽象**：所有决策在编译时完成，运行时无额外成本
- **资源管理**：精心设计确保资源正确释放

这种设计确保了 `std::variant` 在各种使用场景下都能提供合理的异常安全保证，同时保持最佳性能。

#### `get`通过索引或类型访问

首先看通过索引访问的接口，可以看到这里在完成索引范围的检查之后，本质就是对嵌套Union的展开读取

```cpp
template <size_t _Np, typename... _Types>
constexpr variant_alternative_t<_Np, variant<_Types...>> &
get(variant<_Types...> &__v) {
  static_assert(_Np < sizeof...(_Types),
                "The index must be in [0, number of alternatives)");
  if (__v.index() != _Np)  // 检查索引是否匹配
    __throw_bad_variant_access(__v.valueless_by_exception()); // 不匹配则抛异常
  return __detail::__variant::__get<_Np>(__v);
}

template <size_t _Np, typename _Variant>
constexpr decltype(auto) __get(_Variant &&__v) noexcept {
  return __variant::__get_n<_Np>(std::forward<_Variant>(__v)._M_u);
}

template <size_t _Np, typename _Union>
constexpr auto &&__get_n(_Union &&__u) noexcept {
  if constexpr (_Np == 0)
    return std::forward<_Union>(__u)._M_first._M_storage;
  else if constexpr (_Np == 1)
    return std::forward<_Union>(__u)._M_rest._M_first._M_storage;
  else if constexpr (_Np == 2)
    return std::forward<_Union>(__u)._M_rest._M_rest._M_first._M_storage;
  else
    return __variant::__get_n<_Np - 3>(
        std::forward<_Union>(__u)._M_rest._M_rest._M_rest);
}
```

根据类型访问，会先校验类型是 variant 类型参数中的合法类型，然后获取该类型在参数列表中的 idx，转换成根据索引访问

```cpp
template <typename _Tp, typename... _Types>
constexpr _Tp &get(variant<_Types...> &__v) {
  // 要求类型在备选列表中唯一（通过`__exactly_once`验证）
  static_assert(__detail::__variant::__exactly_once<_Tp, _Types...>,
                "T must occur exactly once in alternatives");
  constexpr size_t __n = std::__find_uniq_type_in_pack<_Tp, _Types...>();
  return std::get<__n>(__v);
}
```

#### [`visit`](https://en.cppreference.com/w/cpp/utility/variant/visit2.html)

`visit`是`variant`最强大的功能之一，允许用统一的访问器处理所有可能的活跃类型，实现类似多态的行为。

`visit`的核心是在编译期为所有可能的活跃类型组合生成函数指针表（跳转表），运行时根据当前索引直接调用对应函数。

下面我们来分析其源码实现：

```cpp
template <typename _Visitor, typename... _Variants>
constexpr __detail::__variant::__visit_result_t<_Visitor, _Variants...> // 在编译时推导访问器对 variant 当前活跃类型调用后的返回类型
visit(_Visitor &&__visitor, _Variants &&...__variants) {
  namespace __variant = std::__detail::__variant;
  // 检查是否有valueless状态的variant
  // 通过 as 将参数统一转换为对应的 variant 引用类型，确保类型一致性
  // 同时利用折叠表达式
  if ((__variant::__as(__variants).valueless_by_exception() || ...))
    __throw_bad_variant_access(2);

  using _Result_type =
      __detail::__variant::__visit_result_t<_Visitor, _Variants...>;
  // 提供一个标签类型，用于在 __do_visit 中区分
  using _Tag = __detail::__variant::__deduce_visit_result<_Result_type>;

  if constexpr (sizeof...(_Variants) == 1) {  // 单variant优化路径
    using _Vp = decltype(__variant::__as(std::declval<_Variants>()...));
    // 对 variant 的每一个可能类型，检查访问器的返回类型
    // 确保所有返回类型完全相同
    // 如果类型不一致，触发 static_assert 编译错误
    constexpr bool __visit_rettypes_match =
        __detail::__variant::__check_visitor_results<_Visitor, _Vp>(
            make_index_sequence<variant_size_v<remove_reference_t<_Vp>>>());
    if constexpr (!__visit_rettypes_match) {
      static_assert(__visit_rettypes_match,
                    "std::visit requires the visitor to have the same "
                    "return type for all alternatives of a variant");
      return;
    } else
      return std::__do_visit<_Tag>(std::forward<_Visitor>(__visitor),
                                   static_cast<_Vp>(__variants)...);
  } else
    return std::__do_visit<_Tag>(
        std::forward<_Visitor>(__visitor),
        __variant::__as(std::forward<_Variants>(__variants))...);
}
```

下面查看 `__do_visit` 的实现

```cpp
template <typename _Result_type, typename _Visitor, typename... _Variants>
constexpr decltype(auto) __do_visit(_Visitor &&__visitor,
                                    _Variants &&...__variants) {
  // Get the silly case of visiting no variants out of the way first.
  // 无variant的特殊情况
  if constexpr (sizeof...(_Variants) == 0) {
    if constexpr (is_void_v<_Result_type>)
      return (void)std::forward<_Visitor>(__visitor)();
    else
      return std::forward<_Visitor>(__visitor)();
  } else {
    // 当variant类型数 ≤ 11时，使用switch语句，避免多维数组查找开销
    constexpr size_t __max = 11; // "These go to eleven." 经验值

    // The type of the first variant in the pack.
    using _V0 = typename _Nth_type<0, _Variants...>::type;
    // The number of alternatives in that first variant.
    constexpr auto __n = variant_size_v<remove_reference_t<_V0>>;
    // 多variant或大variant：使用跳转表
    if constexpr (sizeof...(_Variants) > 1 || __n > __max) {
      // Use a jump table for the general case.
      constexpr auto &__vtable =
          __detail::__variant::__gen_vtable<_Result_type, _Visitor &&,
                                            _Variants &&...>::_S_vtable;

      // 对于 visit(f, var1, var2)，访问复杂度 O(1)
      auto __func_ptr = __vtable._M_access(__variants.index()...);
      return (*__func_ptr)(std::forward<_Visitor>(__visitor),
                           std::forward<_Variants>(__variants)...);
    } else // We have a single variant with a small number of alternatives.
    { // 当备选类型数量较少时（≤11），使用`switch-case`替代跳转表，减少间接调用开销
      // A name for the first variant in the pack.
      _V0 &__v0 = [](_V0 &__v, ...) -> _V0 & { return __v; }(__variants...);

      using __detail::__variant::__gen_vtable_impl;
      using __detail::__variant::_Multi_array;
      using _Ma = _Multi_array<_Result_type (*)(_Visitor &&, _V0 &&)>;

#ifdef _GLIBCXX_DEBUG
#define _GLIBCXX_VISIT_UNREACHABLE __builtin_trap
#else
#define _GLIBCXX_VISIT_UNREACHABLE __builtin_unreachable
#endif

#define _GLIBCXX_VISIT_CASE(N)                                                 \
  case N: {                                                                    \
    if constexpr (N < __n) {                                                   \
      return __gen_vtable_impl<_Ma, index_sequence<N>>::__visit_invoke(        \
          std::forward<_Visitor>(__visitor), std::forward<_V0>(__v0));         \
    } else                                                                     \
      _GLIBCXX_VISIT_UNREACHABLE();                                            \
  }

      switch (__v0.index()) {
        _GLIBCXX_VISIT_CASE(0)
        _GLIBCXX_VISIT_CASE(1)
        _GLIBCXX_VISIT_CASE(2)
        _GLIBCXX_VISIT_CASE(3)
        _GLIBCXX_VISIT_CASE(4)
        _GLIBCXX_VISIT_CASE(5)
        _GLIBCXX_VISIT_CASE(6)
        _GLIBCXX_VISIT_CASE(7)
        _GLIBCXX_VISIT_CASE(8)
        _GLIBCXX_VISIT_CASE(9)
        _GLIBCXX_VISIT_CASE(10)
      case variant_npos:
        using __detail::__variant::__variant_cookie;
        using __detail::__variant::__variant_idx_cookie;
        if constexpr (is_same_v<_Result_type, __variant_idx_cookie> ||
                      is_same_v<_Result_type, __variant_cookie>) {
          using _Npos = index_sequence<variant_npos>;
          return __gen_vtable_impl<_Ma, _Npos>::__visit_invoke(
              std::forward<_Visitor>(__visitor), std::forward<_V0>(__v0));
        } else
          _GLIBCXX_VISIT_UNREACHABLE();
      default:
        _GLIBCXX_VISIT_UNREACHABLE();
      }
#undef _GLIBCXX_VISIT_CASE
#undef _GLIBCXX_VISIT_UNREACHABLE
    }
  }
}
```

通过 `__gen_vtable` 通过递归模板实例化在编译时构建一个多维函数指针表

```cpp
template <typename _Result_type, typename _Visitor, typename... _Variants>
struct __gen_vtable {
  using _Array_type =
      _Multi_array<_Result_type (*)(_Visitor, _Variants...),
                   variant_size_v<remove_reference_t<_Variants>>...>;

  static constexpr _Array_type _S_vtable =
      __gen_vtable_impl<_Array_type, std::index_sequence<>>::_S_apply();
};
```

其递归构建过程如下：

```shell
__gen_vtable_impl<Array<2,3>, index_sequence<>>
  ├── __gen_vtable_impl<Array<3>, index_sequence<0>>
  │   ├── __gen_vtable_impl<Array<>, index_sequence<0,0>>
  │   ├── __gen_vtable_impl<Array<>, index_sequence<0,1>> 
  │   └── __gen_vtable_impl<Array<>, index_sequence<0,2>>
  └── __gen_vtable_impl<Array<3>, index_sequence<1>>
      ├── __gen_vtable_impl<Array<>, index_sequence<1,0>>
      ├── __gen_vtable_impl<Array<>, index_sequence<1,1>>
      └── __gen_vtable_impl<Array<>, index_sequence<1,2>>
```

`__gen_vtable_impl` 表生成器

```cpp
// __dimensions...：剩余未处理的维度大小，__indices...：已经确定的索引序列(当前递归路径)
template <typename _Result_type, typename _Visitor, size_t... __dimensions,
          typename... _Variants, size_t... __indices>
struct __gen_vtable_impl<
    _Multi_array<_Result_type (*)(_Visitor, _Variants...), __dimensions...>,
    std::index_sequence<__indices...>> {
  // 确定当前要处理的 variant
  // sizeof...(__indices) 表示已处理的维度数
  // _Nth_type<N, _Variants...> 获取第 N 个 variant 类型
  using _Next = remove_reference_t<
      typename _Nth_type<sizeof...(__indices), _Variants...>::type>;
  using _Array_type =
      _Multi_array<_Result_type (*)(_Visitor, _Variants...), __dimensions...>;

  static constexpr _Array_type _S_apply() {
    _Array_type __vtable{};
    _S_apply_all_alts(__vtable, make_index_sequence<variant_size_v<_Next>>());
    return __vtable;
  }

  template <size_t... __var_indices>
  static constexpr void
  _S_apply_all_alts(_Array_type &__vtable,
                    std::index_sequence<__var_indices...>) {
    if constexpr (__extra_visit_slot_needed<_Result_type, _Next>)
      (_S_apply_single_alt<true, __var_indices>(
           __vtable._M_arr[__var_indices + 1], &(__vtable._M_arr[0])),
       ...);
    else
      (_S_apply_single_alt<false, __var_indices>(
           __vtable._M_arr[__var_indices]),
       ...);
  }

  template <bool __do_cookie, size_t __index, typename _Tp>
  static constexpr void _S_apply_single_alt(_Tp &__element,
                                            _Tp *__cookie_element = nullptr) {
    if constexpr (__do_cookie) {
      // 处理需要无值状态槽位的情况
      __element = __gen_vtable_impl<
          _Tp, std::index_sequence<__indices..., __index>>::_S_apply();
      *__cookie_element = __gen_vtable_impl<
          _Tp, std::index_sequence<__indices..., variant_npos>>::_S_apply();
    } else {
      // 正常情况：递归生成下一层
      auto __tmp_element = __gen_vtable_impl<
          remove_reference_t<decltype(__element)>,
          std::index_sequence<__indices..., __index>>::_S_apply();
      static_assert(is_same_v<_Tp, decltype(__tmp_element)>,
                    "std::visit requires the visitor to have the same "
                    "return type for all alternatives of a variant");
      __element = __tmp_element;
    }
  }
};

// This partial specialization is the base case for the recursion.
// It populates a _Multi_array element with the address of a function
// that invokes the visitor with the alternatives specified by __indices.
// 这个偏特化处理所有维度都已处理完毕的情况，生成实际的函数指针
// 没有 __dimensions... 参数，表示零维数组（单个元素）
// __indices... 包含完整的索引路径
template <typename _Result_type, typename _Visitor, typename... _Variants,
          size_t... __indices>
struct __gen_vtable_impl<_Multi_array<_Result_type (*)(_Visitor, _Variants...)>,
                         std::index_sequence<__indices...>> {
  using _Array_type = _Multi_array<_Result_type (*)(_Visitor, _Variants...)>;

  // 根据索引获取 variant 中的值，或返回特殊标记
  template <size_t __index, typename _Variant>
  static constexpr decltype(auto)
  __element_by_index_or_cookie(_Variant &&__var) noexcept {
    if constexpr (__index != variant_npos)
      return __variant::__get<__index>(std::forward<_Variant>(__var));
    else
      return __variant_cookie{};
  }

  // 核心调用函数
  static constexpr decltype(auto) __visit_invoke(_Visitor &&__visitor,
                                                 _Variants... __vars) {
    if constexpr (is_same_v<_Result_type, __variant_idx_cookie>)
      // For raw visitation using indices, pass the indices to the visitor
      // and discard the return value:
      // /带索引的原始访问（无返回值）
      std::__invoke(std::forward<_Visitor>(__visitor),
                    __element_by_index_or_cookie<__indices>(
                        std::forward<_Variants>(__vars))...,
                    integral_constant<size_t, __indices>()...);
    else if constexpr (is_same_v<_Result_type, __variant_cookie>)
      // For raw visitation without indices, and discard the return value:
      // 不带索引的原始访问（无返回值）
      std::__invoke(std::forward<_Visitor>(__visitor),
                    __element_by_index_or_cookie<__indices>(
                        std::forward<_Variants>(__vars))...);
    else if constexpr (_Array_type::__result_is_deduced::value)
      // For the usual std::visit case deduce the return value:
      // 自动推导返回类型
      return std::__invoke(std::forward<_Visitor>(__visitor),
                           __element_by_index_or_cookie<__indices>(
                               std::forward<_Variants>(__vars))...);
    else // for std::visit<R> use INVOKE<R>
      // 显式指定返回类型
      return std::__invoke_r<_Result_type>(
          std::forward<_Visitor>(__visitor),
          __variant::__get<__indices>(std::forward<_Variants>(__vars))...);
  }

  static constexpr auto _S_apply() {
    if constexpr (_Array_type::__result_is_deduced::value) {
      constexpr bool __visit_ret_type_mismatch =
          !is_same_v<typename _Result_type::type,
                     decltype(__visit_invoke(std::declval<_Visitor>(),
                                             std::declval<_Variants>()...))>;
      if constexpr (__visit_ret_type_mismatch) {
        struct __cannot_match {};
        return __cannot_match{};
      } else
        return _Array_type{&__visit_invoke};
    } else
      return _Array_type{&__visit_invoke};
  }
};
```

其生成的结构是一个多维函数指针表

```cpp
// 这是一个编译时生成的多维数组，用于存储所有可能的访问路径
template <typename _Tp, size_t... _Dimensions> 
struct _Multi_array;

// 对于 visit(visitor, variant<int, char>, variant<float, double, string>)
// 会实例化为：
_Multi_array<void(*)(Visitor, V1&&, V2&&), 2, 3>
```

##### 例子

直接看源码似乎有些晦涩，。让我们通过一个具体例子来理解：

```cpp
// 假设调用：visit(visitor, var1, var2)
// 其中：var1: variant<int, string> (2种类型)
//      var2: variant<double, float> (2种类型)
```

首先是第一层的初始调用

```cpp
// 实例化 __gen_vtable
using VTable = __gen_vtable<void, Visitor&&, Var1&&, Var2&&>;

// _Array_type 推导为：
_Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2, 2>
```

然后 __gen_vtable 会利用 __gen_vtable_impl 进行递归的展开

```cpp
// 进入 __gen_vtable_impl 主模板
__gen_vtable_impl<
    _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2, 2>, 
    std::index_sequence<>>
```

此时：

- `sizeof...(__indices) = 0` （还没有处理任何维度）
- `_Next = Var1` （第一个variant类型）

展开 `_S_apply_all_alts`：

```cpp
// 为第一个variant的每个可能索引生成子表
_S_apply_all_alts(__vtable, std::index_sequence<0, 1>{});

// 展开为：
_S_apply_single_alt<false, 0>(__vtable._M_arr[0]);
_S_apply_single_alt<false, 1>(__vtable._M_arr[1]);
```

处理第一个维度，对于索引0：

```cpp
_S_apply_single_alt<false, 0>(__vtable._M_arr[0]) {
    auto __tmp_element = __gen_vtable_impl<
        _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2>,
        std::index_sequence<0>
    >::_S_apply();
    __vtable._M_arr[0] = __tmp_element;
}
```

对于索引1：

```cpp
_S_apply_single_alt<false, 1>(__vtable._M_arr[1]) {
    auto __tmp_element = __gen_vtable_impl<
        _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2>,  
        std::index_sequence<1>
    >::_S_apply();
    __vtable._M_arr[1] = __tmp_element;
}
```

处理第二个维度，现在进入更深层的递归，以 `index_sequence<0>` 为例：

```cpp
__gen_vtable_impl<
    _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2>,
    std::index_sequence<0>>
```

此时：

- `sizeof...(__indices) = 1` （已处理1个维度）
- `_Next = Var2` （第二个variant类型）

展开第二个维度的 `_S_apply_all_alts`：

```cpp
// 为第二个variant的每个可能索引生成子表
_S_apply_all_alts(__vtable, std::index_sequence<0, 1>{});

// 展开为：
_S_apply_single_alt<false, 0>(__vtable._M_arr[0]);
_S_apply_single_alt<false, 1>(__vtable._M_arr[1]);
```

现在进入最深层，生成实际的函数指针：

```cpp
// 对于组合 (0,0)
__gen_vtable_impl<
    _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&)>,
    std::index_sequence<0, 0>>
```

这是**基本情况特化**，调用 `_S_apply()` 返回：

```cpp
_Multi_array<void(*)(Visitor&&, Var1&&, Var2&&)>{
    &__visit_invoke  // 指向具体实现的函数指针
}
```

`__visit_invoke` 的具体实现：

```cpp
static constexpr decltype(auto) __visit_invoke(Visitor&& __visitor,
                                               Var1&& __var1, Var2&& __var2) {
    return std::__invoke(std::forward<Visitor>(__visitor),
                         __get<0>(std::forward<Var1>(__var1)),  // int
                         __get<0>(std::forward<Var2>(__var2))); // double
}
```

经过完整的递归展开，生成如下的二维函数指针表：

```cpp
_Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2, 2> vtable = {
    _M_arr: [
        // [0][*] - 对应 var1 持有 int
        _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2> {
            _M_arr: [
                &visit_impl<int, double>,   // [0][0]
                &visit_impl<int, float>     // [0][1]
            ]
        },
        // [1][*] - 对应 var1 持有 string  
        _Multi_array<void(*)(Visitor&&, Var1&&, Var2&&), 2> {
            _M_arr: [
                &visit_impl<string, double>, // [1][0]
                &visit_impl<string, float>   // [1][1]
            ]
        }
    ]
};
```

如果 variant 可能处于无值状态，会生成额外的槽位：

```cpp
if constexpr (__extra_visit_slot_needed<_Result_type, _Next>)
    // 为无值状态生成额外槽位
    _S_apply_single_alt<true, __var_indices>(...);
```

这会为每个维度添加一个处理 `variant_npos` 的特殊情况。

返回类型推导

```cpp
if constexpr (_Array_type::__result_is_deduced::value) {
    // 自动推导返回类型
    return std::__invoke(visitor, args...);
} else {
    // 显式指定返回类型  
    return std::__invoke_r<_Result_type>(visitor, args...);
}
```

编译时生成表后，运行时访问极其高效：

```cpp
// 访问表元素：O(1) 时间复杂度
auto func_ptr = vtable._M_access(var1.index(), var2.index());

// 调用对应的处理函数
func_ptr(std::forward<Visitor>(visitor), 
         std::forward<Var1>(var1), 
         std::forward<Var2>(var2));
```

回过头来看，这样设计的优势在于：

- 所有函数指针在编译期确定，零运行时初始化开销
- 递归深度是可控的

```cpp
// 递归基：当没有更多维度时
template <typename _Result_type, typename _Visitor, typename... _Variants,
          size_t... __indices>
struct __gen_vtable_impl<_Multi_array<_Result_type (*)(_Visitor, _Variants...)>,
                         std::index_sequence<__indices...>>
```

- 每个索引组合都有对应的类型安全函数，编译时验证所有可能的类型组合
- 生成的多维表可能很大，但访问是 O(1)，适合variant类型数量适中的场景

## 一些引申的思考

### 为什么`struct _Uninitialized`需要针对`std::is_trivially_destructible_v`进行特化

这是因为，**对于包含非平凡类型的联合体，其默认析构函数会被隐式删除，需要程序员显式定义联合体的析构函数并手动调用活跃成员的析构函数**。

1. **特殊成员函数的隐式删除**：自C++11起，如果联合体（union）包含具有非平凡（non-trivial）析构函数的成员，那么联合体自身的析构函数会**被隐式删除（implicitly deleted）**。这意味着编译器不会为其生成默认的析构函数。See [`Union`](https://en.cppreference.com/w/cpp/language/union.html)
  
> 在C++11之前，联合体不能包含具有非平凡特殊成员函数（如析构函数）的类型。
>
> If a union contains a non-static data member with a non-trivial special member function, the corresponding special member function of the union may be defined as deleted, see the corresponding special member function page for details. 

2. **程序员的责任**：一旦联合体的析构函数被隐式删除，**程序员必须手动定义联合体的析构函数**，并在其中**显式调用当前活跃成员的析构函数**。如果程序员没有提供，那么尝试析构该联合体对象就会导致编译错误。

3. **底层原因**：联合体所有成员共享同一块内存地址。在任一时刻，只有一个成员是“活跃”的（即被初始化的）。由于编译器无法在编译期确定哪个成员是活跃的，它也就无法在联合体析构时自动插入对所有可能成员的析构函数调用。因此，这个责任就交给了程序员。

下面是一个简单的例子来说明这个问题：

```cpp
#include <iostream>
#include <string>

union MyUnion {
    std::string str; // std::string 有非平凡的析构函数
    int number;
    // 默认析构函数被隐式删除，因为 std::string 有非平凡析构函数
    // ~MyUnion() = delete; (由编译器隐式声明)
};

int main() {
    MyUnion u;
    new (&u.str) std::string("Hello"); // 使用 placement new 初始化 string 成员
    u.str.~basic_string(); // 必须手动调用 string 的析构函数
    // 如果此处没有手动调用 u.str 的析构函数，并且 MyUnion 也没有自定义析构函数，
    // 那么 u.str 的析构函数将不会被调用，可能导致内存泄漏。
    return 0;
}
```

在这个例子中，因为 `MyUnion` 包含了一个 `std::string` 成员（它拥有非平凡的析构函数），所以 `MyUnion` 的默认析构函数会被隐式删除。如果在 `main` 函数中没有显式调用 `u.str.~basic_string()`，那么 `std::string` 的析构函数就不会被调用，从而导致内存泄漏。

### 内存分析

从 `_Variant_storage` 的实现上可以看到，`std::variant` 的内存布局主要由两部分组成：

1. 存储区域：用于存储当前活动类型的值，大小至少为最大可能类型的大小
2. 索引：用于跟踪当前持有的类型，通常是一个整数

因此，`std::variant` 的总体大小约为：`sizeof(std::variant<Types...>) ≈ max(sizeof(Types...)) + sizeof(index_type)`

```cpp
template <bool __trivially_destructible, typename... _Types>
struct _Variant_storage {
  _Variadic_union<__trivially_destructible, _Types...> _M_u;  // 存储备选类型的联合体
  using __index_type = __select_index<_Types...>;  // 最小化的索引类型（如char/short）
  __index_type _M_index;  // 记录当前活跃类型的索引（variant_npos表示无值）
```

那么 `__index_type` 又是如何选定的呢？

```cpp
template <typename... _Types>
using __select_index =
    typename __select_int::_Select_int_base<sizeof...(_Types), unsigned char,
                                            unsigned short>::type::value_type;

// libstdc++-v3/include/bits/parse_numbers.h
namespace __select_int {
  ……
  template<unsigned long long _Val, typename _IntType, typename... _Ints>
    struct _Select_int_base<_Val, _IntType, _Ints...>
    : __conditional_t<(_Val <= __gnu_cxx::__int_traits<_IntType>::__max),
        integral_constant<_IntType, (_IntType)_Val>,
        _Select_int_base<_Val, _Ints...>>
    { };
  ……
}
```

可以看到，index_type 的大小根据类型数量自动选择：

- 如果类型数量 ≤ 256，使用 uint8_t
- 如果类型数量 ≤ 65536，使用 uint16_t
- 理论上可以支持更的整数类型，不过在gcc的实现中最大只到`unsigned short`

索引值从0开始，对应于模板参数列表中的类型顺序。特殊值 variant_npos（通常是 size_t 的最大值）用于表示"valueless by exception"状态。

```cpp
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
```

对齐要求遵循最严格的类型对齐要求，可能导致额外的填充字节：

`alignof(std::variant<Types...>) = max(alignof(Types...))`

### 与其他工具的对比

`std::variant`与传统`union`比较。传统`union`是C风格联合体，仅提供原始内存共享能力。

| 维度         | `std::variant`                          | 传统`union`                          |
|--------------|----------------------------------------|--------------------------------------|
| 类型安全     | 完全类型安全（编译期检查）| 不安全，需手动跟踪当前存储类型，易引发未定义行为 |
| 非POD类型支持 | 支持任何可复制构造的类型（如`std::string`、自定义类） | 仅支持POD类型，非POD类型需手动管理构造/析构 |
| 默认值       | 默认构造为第一个类型                     | 无定义初始状态，需手动初始化           |
| 析构行为     | 自动调用当前类型的析构函数               | 需手动调用析构函数，否则导致资源泄漏   |
| 内存开销     | 额外存储类型索引（通常1~2字节）| 仅存储值，无额外开销                  |
| 使用复杂度   | 提供`std::visit`、`std::get`等高级API，易于安全使用 | 低级API，需手动维护类型状态，容易出错  |

`std::variant`与`boost::variant`比较

| 维度         | `std::variant`                          | `boost::variant`                      |
|--------------|----------------------------------------|--------------------------------------|
| 无值状态     | 支持`valueless_by_exception`（异常时可能无值） | 始终持有一个值（`never-empty`保证）|
| 递归变体     | 不直接支持递归类型（如`variant<int, variant<int, string>>`） | 通过`recursive_wrapper`支持递归变体   |
| 访问API      | `std::visit`（访问者模式）、`std::get`       | `apply_visitor`、`get`                |
| 引用支持     | 不支持存储引用类型                       | 可通过`wrapper`支持引用类型           |
| 性能         | 通常更优化（标准库原生支持，编译期优化多）| 稍慢但功能更丰富（如更多辅助工具）|
| 标准支持     | C++17标准库原生支持                      | 需额外依赖Boost库                    |

`std::variant`与`std::any`比较。`std::any`是C++17引入的**动态类型容器**，可存储任意类型（运行时类型检查）。

| 维度         | `std::variant`                          | `std::any`                           |
|--------------|----------------------------------------|--------------------------------------|
| 类型安全     | 编译期已知类型集（静态类型检查）| 任意类型，运行时类型检查（动态安全）|
| 内存模型     | 通常在栈上存储，无动态分配（除非类型过大） | 可能需要动态分配（存储大类型或非平凡类型时） |
| 性能         | 更高效（无类型擦除开销，编译期优化）| 存在类型擦除开销，运行时类型检查有性能损耗 |
| 灵活性       | 有限的预定义类型集（需在编译期指定所有可能类型） | 完全动态，可存储任意类型             |
| 使用场景     | 已知类型集的多态场景（如配置项、状态机）| 完全动态类型系统（如脚本引擎、序列化）|

适用场景

- **选择`std::variant`**：当类型集在编译期确定，需类型安全、低开销的多态存储时（如状态机、配置解析）。
- **选择传统`union`**：仅在需极致内存开销且处理POD类型的底层场景（如硬件驱动）。
- **选择`boost::variant`**：若项目依赖Boost且需递归变体、引用支持等高级功能，且无需标准库原生支持时。
- **选择`std::any`**：当需完全动态的类型存储（如插件系统、动态脚本交互），可接受运行时类型检查开销时。

## 用法示例

### 构造

```cpp
// 默认构造（构造第一个类型的默认值）
std::variant<int, std::string> v1;  // 包含int(0)

// 从特定类型构造
std::variant<int, std::string> v2 = "hello";  // 包含std::string("hello")

// 使用in_place_type直接构造
std::variant<int, std::string> v3(std::in_place_type<std::string>, "world");

// 使用in_place_index按索引构造
std::variant<int, std::string> v4(std::in_place_index<1>, "world");
```

### 访问

```cpp
// 使用get获取值（不安全，类型错误会抛出异常）
try {
    std::cout << std::get<int>(v1) << std::endl;        // 按类型访问
    std::cout << std::get<0>(v1) << std::endl;          // 按索引访问
} catch (const std::bad_variant_access& e) {
    std::cerr << "类型访问错误: " << e.what() << std::endl;
}

// 使用get_if安全访问（返回指针，类型错误返回nullptr）
if (auto pval = std::get_if<std::string>(&v2)) {
    std::cout << *pval << std::endl;
} else {
    std::cout << "v2不包含string" << std::endl;
}

// 检查当前类型
if (std::holds_alternative<int>(v1)) {
    std::cout << "v1包含int" << std::endl;
}
```

### 修改

```cpp
// 赋值
v1 = "hello";  // 现在包含std::string

// 原位构造
v1.emplace<int>(42);  // 现在包含int(42)
v1.emplace<0>(42);    // 等效写法
```

### 通过`overload`辅助类实现类switch-case语义

```cpp
// https://godbolt.org/z/r6a9Y6jxh
// overloaded辅助类
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>; // C++17指引推导

// 使用示例
std::variant<int, std::string> v = "hello";
std::visit(overloaded {
    [](int i) { std::cout << "整数: " << i << std::endl; },
    [](const std::string& s) { std::cout << "字符串: " << s << std::endl; }
}, v);
```

### 结合`overload`实现状态机

```cpp
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
```

### 返回统一类型

```cpp
// https://godbolt.org/z/P4bM75jda
auto result = std::visit([](const auto& val) -> std::string {
    using std::to_string;
    if constexpr (std::is_same_v<std::decay_t<decltype(val)>, std::string>)
        return val;
    else
        return to_string(val);
}, v);
```

### 多个`variant`组合访问

```cpp
// https://godbolt.org/z/jnoYszsTb
#include <variant>
#include <iostream>
#include <assert.h>

int main() {
    std::variant<int, float> v1 = 42;
    std::variant<std::string, bool> v2 = "hello";

    std::visit([](auto&& a, auto&& b) {
      std::cout << a << " - " << b << std::endl;
    }, v1, v2);

    return 0;
}

// 42 - hello
```

### 通过`variant`返回不同类型的值

```cpp
// 函数可以返回不同类型
std::variant<std::string, std::vector<int>, std::error_code> 
parse_input(const std::string& input) {
    if (input.empty())
        return std::error_code(EINVAL, std::generic_category());
    
    if (input[0] == '[')
        return std::vector<int>{1, 2, 3}; // 解析为数组
    
    return input; // 返回原始字符串
}
```

### 利用`variant`实现异构容器

```cpp
// 可以存储不同类型的集合
std::vector<std::variant<int, std::string, double>> collection;
collection.push_back(42);
collection.push_back("Hello");
collection.push_back(3.14);

// 处理所有元素
for (const auto& item : collection) {
    std::visit([](const auto& val) { std::cout << val << std::endl; }, item);
}
```
