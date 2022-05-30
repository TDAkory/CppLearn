# What Happens when we use `std::make_shared<>`

> /usr/include/c++/8/bits/shared_ptr.h 
> /usr/include/c++/8/bits/shared_ptr_base.h

### 1. `std::make_shared` 

```cpp
/**
   *  @brief  Create an object that is owned by a shared_ptr.
   *  @param  __args  Arguments for the @a _Tp object's constructor.
   *  @return A shared_ptr that owns the newly created object.
   *  @throw  std::bad_alloc, or an exception thrown from the
   *          constructor of @a _Tp.
   */
  template<typename _Tp, typename... _Args>
    inline shared_ptr<_Tp>
    make_shared(_Args&&... __args)
    {
      typedef typename std::remove_cv<_Tp>::type _Tp_nc;
      return std::allocate_shared<_Tp>(std::allocator<_Tp_nc>(),
                      std::forward<_Args>(__args)...);
    }
```

### 2. `std::make_shared` 调用 `std::allocate_shared`

```cpp
/**
   *  @brief  Create an object that is owned by a shared_ptr.
   *  @param  __a     An allocator.
   *  @param  __args  Arguments for the @a _Tp object's constructor.
   *  @return A shared_ptr that owns the newly created object.
   *  @throw  An exception thrown from @a _Alloc::allocate or from the
   *          constructor of @a _Tp.
   *
   *  A copy of @a __a will be used to allocate memory for the shared_ptr
   *  and the new object.
   */
  template<typename _Tp, typename _Alloc, typename... _Args>
    inline shared_ptr<_Tp>
    allocate_shared(const _Alloc& __a, _Args&&... __args)
    {
      return shared_ptr<_Tp>(_Sp_alloc_shared_tag<_Alloc>{__a},
                 std::forward<_Args>(__args)...);
    }
```

#### 2.1 `shared_ptr` 继承自 `__shared_ptr`

```cpp
template<typename _Tp>
    class shared_ptr : public __shared_ptr<_Tp>
    {...}
```

#### 2.2 `_Sp_alloc_shared_tag` 是定义在基类中的，即声明一个`_Tp`的内存分配器

```cpp
 template<typename _Alloc>
    struct _Sp_alloc_shared_tag
    {
      const _Alloc& _M_a;
    };
```

#### 2.3 检查`shared_ptr`的构造函数，可以看到`shared_ptr`可以通过裸指针、`unique_ptr`、`weak_ptr`、`shared_ptr`来构造，但并没有第一个参数是分配器的公有构造函数。

#### 2.4 可以看到 `allocate_shared` 被声明为友元函数，可以访问`shared_ptr`的私有构造，最终会调用基类的构造函数

```cpp
  private:
    // This constructor is non-standard, it is used by allocate_shared.
    template<typename _Alloc, typename... _Args>
    shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args)
    : __shared_ptr<_Tp>(__tag, std::forward<_Args>(__args)...)
    { }

    template<typename _Yp, typename _Alloc, typename... _Args>
	friend shared_ptr<_Yp>
	allocate_shared(const _Alloc& __a, _Args&&... __args);
```

### 3. 在基类中，调用其构造函数

```cpp
    // This constructor is non-standard, it is used by allocate_shared.
    template<typename _Alloc, typename... _Args>
    __shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args)
    : _M_ptr(), _M_refcount(_M_ptr, __tag, std::forward<_Args>(__args)...)
    { _M_enable_shared_from_this_with(_M_ptr); }
```

#### 3.1 `_M_ptr` 是共享指针的模板实参类型指针

```cpp
using element_type = typename remove_extent<_Tp>::type;
...
element_type*	   _M_ptr;         // Contained pointer.
```

#### 3.2 `_M_refcount` 是 `__shared_count` 类型的变量，记录引用计数，同时内存分配也是在这里发生的

```cpp
  template<_Lock_policy _Lp>
    class __shared_count {...};

__shared_count<_Lp>  _M_refcount;    // Reference counter.
```

##### 3.2.1 `__shared_count` 的构造

```cpp
template<typename _Tp, typename _Alloc, typename... _Args>
	__shared_count(_Tp*& __p, _Sp_alloc_shared_tag<_Alloc> __a,
		       _Args&&... __args)
	{
	  typedef _Sp_counted_ptr_inplace<_Tp, _Alloc, _Lp> _Sp_cp_type;
	  typename _Sp_cp_type::__allocator_type __a2(__a._M_a);
	  auto __guard = std::__allocate_guarded(__a2);
	  _Sp_cp_type* __mem = __guard.get();   // 返回上一步分配的内存指针
	  auto __pi = ::new (__mem) 
	    _Sp_cp_type(__a._M_a, std::forward<_Args>(__args)...);  // placement_new 构造新对象
	  __guard = nullptr;
	  _M_pi = __pi;
	  __p = __pi->_M_ptr();
	}
```
`__allocate_guarded` 是定义在 `allocated_ptr.h` 中的一个helper function，它调用了`allocator_traits<_Alloc>::allocate(...)`，这里的模板参数`_Alloc`是所指定的内存分配器，如果用户没有指定，默认情况下它是标准的allocator，它的静态成员函数allocate最终调用了`operator new`函数在堆上分配了内存。

#### 3.3 `_M_enable_shared_from_this_with` 是一个模板函数

```cpp
template<typename _Yp, typename _Yp2 = typename remove_cv<_Yp>::type>
	typename enable_if<__has_esft_base<_Yp2>::value>::type
	_M_enable_shared_from_this_with(_Yp* __p) noexcept
	{
	  if (auto __base = __enable_shared_from_this_base(_M_refcount, __p))
	    __base->_M_weak_assign(const_cast<_Yp2*>(__p), _M_refcount);
	}
```
