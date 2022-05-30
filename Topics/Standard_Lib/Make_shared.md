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

### 4. std::shared_ptr的内存结构

首先shared_ptr大概包含以下数据单元：指向data field的element_type *类型的指针，以及一个间接的包含了_M_use_count，_M_weak_count的__shared_count（在某些情况下它可能还包含一个deletor对象和一个allocator对象，这一区域被称为control block，__shared_count中包含一个指向这个control block的指针）。

![std::shared_ptr的内存结构](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/20220530222709.png)

在__shared_ptr里，会通过萃取技术为_Sp_counted_ptr_inplace开辟出一块内存，其中_Sp_counted_ptr_inplace唯一的数据成员是_Impl类型的_M_impl。下面来看_Sp_counted_ptr_inplace中直接以及间接包含的信息：

* 由于_Sp_counted_ptr_inplace的父类是_Sp_counted_base，而_Sp_counted_base里有_M_use_count和_M_weak_count两个成员，因此_Sp_counted_ptr_inplace间接的包含了_M_use_count和_M_weak_count。
* 由于_M_impl继承自 _Sp_ebo_helper<0, _Alloc>，而 _Sp_ebo_helper<0, _Alloc>是一个为了使用空基类优化（EBCO）而引入的辅助类，因此_Sp_counted_ptr_inplace还间接的包含了一个allocator对象。
* 由于_M_impl还有一个__gnu_cxx::aligned_buffer<_Tp> _M_storage成员，而__gnu_cxx::aligned_buffer<_Tp>包含的是一个大小和经过内存对其后的_Tp的大小相同的char数组，其目的是用来存储_Tp，因此_Sp_counted_ptr_inplace还间接包含了一个_Tp。

上述1和2对应于control block，3对应于data fiels。因此在//call stack #0中，通过类型萃取std::__allocate_guarded为_Sp_counted_ptr_inplace开辟内存，就等于同时为data field和control block开辟了内存。这也正是std::make_shared的精妙所在。