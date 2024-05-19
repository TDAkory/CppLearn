# [Coroutines](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

- [Coroutines](#coroutines)
  - [Refs](#refs)
  - [Basic Ideas](#basic-ideas)
    - [What does the Coroutines TS give us?](#what-does-the-coroutines-ts-give-us)
    - [Explain in cppreference](#explain-in-cppreference)
    - [Understanding](#understanding)
      - [什么是协程](#什么是协程)
      - [协程的状态](#协程的状态)
      - [协程的挂起](#协程的挂起)
      - [协程的返回值](#协程的返回值)
      - [协程体的执行](#协程体的执行)
      - [协程体的返回值](#协程体的返回值)
      - [协程体抛出异常](#协程体抛出异常)
  - [`Awaitable` Interface](#awaitable-interface)
    - [Awaiters and Awaitables: Explaining operator `co_await`](#awaiters-and-awaitables-explaining-operator-co_await)
    - [Obtaining the Awaiter](#obtaining-the-awaiter)
    - [Awaiting the Awaiter](#awaiting-the-awaiter)
    - [Coroutine Handles](#coroutine-handles)
  - [Synchronisation-free async code](#synchronisation-free-async-code)
    - [Comparison to Stackful Coroutines](#comparison-to-stackful-coroutines)
    - [Avoiding memory allocations](#avoiding-memory-allocations)
    - [Example](#example)
  - [`Promise` Interface](#promise-interface)
    - [分配协程帧](#分配协程帧)
      - [Customising coroutine frame memory allocaiton](#customising-coroutine-frame-memory-allocaiton)
    - [拷贝参数到协程帧](#拷贝参数到协程帧)
    - [Constructing the promise object](#constructing-the-promise-object)
    - [Obtaining the return object](#obtaining-the-return-object)
    - [The initial-suspend point](#the-initial-suspend-point)
    - [Returning to the](#returning-to-the)
    - [Returning from the coroutine using `co_return`](#returning-from-the-coroutine-using-co_return)
    - [Handling exceptions that propagate out of the coroutine body](#handling-exceptions-that-propagate-out-of-the-coroutine-body)
    - [The final-suspend point](#the-final-suspend-point)
    - [How the compiler chooses the promise type](#how-the-compiler-chooses-the-promise-type)
    - [Identifying a specific cotoutine activation frame](#identifying-a-specific-cotoutine-activation-frame)
    - [自定义 co\_await 的行为](#自定义-co_await-的行为)
    - [自定义 co\_yield 的行为](#自定义-co_yield-的行为)
  - [Understanding Symmetric Transfer](#understanding-symmetric-transfer)
    - [how a task coroutine works](#how-a-task-coroutine-works)
    - [TODO 一大串没看懂 。。。。。。](#todo-一大串没看懂-)


## Refs

- [Coroutine Theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory)
- [C++ Coroutines: Understanding operator co_await](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)
- [C++ Coroutines: Understanding the promise type](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type)
- [C++ Coroutines: Understanding Symmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)
- [C++ Coroutines: Understanding the Compiler Transform](https://lewissbaker.github.io/2022/08/27/understanding-the-compiler-transform)

- [Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)
- [Coroutines and Reference Parameters](https://toby-allsopp.github.io/2017/04/22/coroutines-reference-params.html)

- [Yet Another C++ Coroutine Tutorial](https://theshoemaker.de/posts/yet-another-cpp-coroutine-tutorial)
- [andreasbuhr cppcoro](https://github.com/andreasbuhr/cppcoro)
- [C++20 Coroutines and io_uring](https://pabloariasal.github.io/2022/11/12/couring-1/)
- [渡劫 C++ 协程](https://www.bennyhuo.com/2022/03/09/cpp-coroutines-01-intro/)

## Basic Ideas

### What does the Coroutines TS give us?

* Three new language keywords: `co_await`, `co_yield` and `co_return`
  * `co_yield expr` 等价于 `co_await promise.yield_value(expr)`
* Several new concepts:
  * `coroutine_state` : 记录协程状态，C++ 协程会在开始执行时的第一步就使用 operator new 来开辟一块内存来存放
  * `coroutine_handle<P>` : 协程的唯一标识，用于恢复执行或者销毁协程帧
  * `coroutine_traits<Ts...>`
  * `promise` : 协程的状态信息，用于存储协程的状态信息，包括协程的返回值、协程的异常信息等
  * `awaiter` : 用于控制 `co_await` 表达式的语义，包括是否挂起协程、在挂起后执行某些逻辑、在恢复时执行某些逻辑来处理 `co_await` 的返回值。标准库当中提供了两个非常简单直接的等待体
    * `suspend_always` : 总是挂起
    * `suspend_never` : 总是不挂起
* A general mechanism that library writers can use to interact with coroutines and customise their behaviour.
* A language facility that makes writing asynchronous code a whole lot easier!

协程提案并没有明确规定协程的语义（semantics），而是定义了一个通用的机制，使得库代码可以通过实现符合特定接口的类型，来定制化协程的行为。

提案定义了两类主要的接口：`Promise` `Awaitable`

- `Promise`接口定义了定制协程自身行为的方法：库作者可以定制coroutine在 `called`、`return`、`co_await`、`co_yield` 时的行为

`Awaitable`接口定义了控制`co_await`语义的方法：当一个变量是 `co_await`ed，那么编译器会生成一系列可指定的针对 `awaitable` 变量的方法，这些方法包括：是否挂起协程、在挂起后执行某些逻辑、在恢复时执行某些逻辑来处理`co_await`的返回值。

### Explain in cppreference

The unary operator co_await suspends a coroutine and returns control to the caller. Its operand is an expression that either (1) is of **a class type that defines a member operator `co_await`** or may **be passed to a non-member operator `co_await`**, or (2) is **convertible to such a class type by means of the current coroutine's `Promise::await_transform`**.

```cpp
co_await expr
```

First, expr is converted to an awaitable as follows:

- if expr is produced by an initial suspend point, a final suspend point, or a yield expression, the awaitable is expr, as-is.
- otherwise, if the current coroutine's Promise type has the member function await_transform, then the awaitable is promise.await_transform(expr).
- otherwise, the awaitable is expr, as-is.

Then, the awaiter object is obtained, as follows:

- if overload resolution for operator co_await gives a single best overload, the awaiter is the result of that call:
  - awaitable.operator co_await() for member overload,
  - operator co_await(static_cast<Awaitable&&>(awaitable)) for the non-member overload.
- otherwise, if overload resolution finds no operator co_await, the awaiter is awaitable, as-is.
- otherwise, if overload resolution is ambiguous, the program is ill-formed.

### Understanding

#### 什么是协程

协程就是一段可以挂起（suspend）和恢复（resume）的程序，一般而言，就是一个支持挂起和恢复的函数。

#### 协程的状态

协程挂起时，我们需要记录函数执行的位置，C++ 协程会在开始执行时的第一步就使用 operator new 来开辟一块内存来存放这些信息，这块内存或者说这个对象又被称为协程的状态（coroutine state）。

#### 协程的挂起

C++ 通过 co_await 表达式来处理协程的挂起，表达式的操作对象则为等待体（awaiter）

**实现了`await_ready`、`await_suspend`、 `await_resume`接口的类型被称为`Awaiter`**

1.  `await_ready` 返回 bool 类型，如果返回 true，则表示已经就绪，无需挂起；否则表示需要挂起。

```cpp
struct suspend_never {
  constexpr bool await_ready() const noexcept { return true; }
  ...
};

struct suspend_always {
  constexpr bool await_ready() const noexcept { return false; }
  ...
};
```  

2. `await_ready` 返回 false 时，协程就挂起了。这时候协程的局部变量和挂起点都会被存入协程的状态当中，`await_suspend` 被调用到。

```cpp
??? await_suspend(std::coroutine_handle<> coroutine_handle);
```

参数 `coroutine_handle` 用来表示当前协程，`await_suspend` 函数的返回值类型对应着不同的行为：
  * 返回 void 类型或者返回 true，表示当前协程挂起之后将执行权还给当初调用或者恢复当前协程的函数。
  * 返回 false，则恢复执行当前协程。注意此时不同于 await_ready 返回 true 的情形，此时协程已经挂起，await_suspend 返回 false 相当于挂起又立即恢复。
  * 返回其他协程的 coroutine_handle 对象，这时候返回的 coroutine_handle 对应的协程被恢复执行。
  * 抛出异常，此时当前协程恢复执行，并在当前协程当中抛出异常。

3. 协程恢复执行之后，等待体的 await_resume 函数被调用。同样地，await_resume 的返回值类型也是不限定的，返回值将作为 co_await 表达式的返回值。

```cpp
??? await_resume()；
```

#### 协程的返回值

在 C++ 当中，一个函数的返回值类型如果是符合协程的规则的类型，那么这个函数就是一个协程。

这个协程的规则，就是返回值类型能够实例化如下模板。也就是说，返回值类型 _Ret 能够找到一个类型 _Ret::promise_type 与之相匹配。这个 promise_type 既可以是直接定义在 _Ret 当中的类型，也可以通过 using 指向已经存在的其他外部类型。

```cpp
template <class _Ret, class = void>
struct _Coroutine_traits {};

template <class _Ret>
struct _Coroutine_traits<_Ret, void_t<typename _Ret::promise_type>> {
    using promise_type = typename _Ret::promise_type;
};

template <class _Ret, class...>
struct coroutine_traits : _Coroutine_traits<_Ret> {};

struct Result {
  struct promise_type {

    Result get_return_object() {
      // 创建 Result 对象
      return {};
    }

    ...
  };
};
```

此外，Result对象的创建，应当由 promise_type 通过 get_return_object 接口来获得。不同于一般的函数，协程的返回值并不是在返回之前才创建，而是在协程的状态创建出来之后马上就创建的。也就是说，协程的状态被创建出来之后，会立即构造 promise_type 对象，进而调用 get_return_object 来创建返回值对象。

promise_type 类型的构造函数参数列表如果与协程的参数列表一致，那么构造 promise_type 时就会调用这个构造函数。否则，就通过默认无参构造函数来构造 promise_type。

#### 协程体的执行

在协程的返回值被创建之后，协程体就要被执行了。

1. 为了便于扩展，协程体执行第一步就是调用 `co_await promise.initial_suspend()`，其返回值是一个awaiter，可以通过这个awaiter来实现协程的调度
2. 然后执行协程体，协程体当中会存在 co_await、co_yield、co_return 三种协程特有的调用
3. 当协程执行完成或者抛出异常之后会先清理局部变量，接着调用 final_suspend 来方便开发者自行处理其他资源的销毁逻辑。final_suspend 也可以返回一个等待体使得当前协程挂起，但之后当前协程应当通过 coroutine_handle 的 destroy 函数来直接销毁，而不是 resume。

#### 协程体的返回值

```cpp
// 对于返回一个值的情况，需要在 promise_type 当中定义一个函数
struct Result {
  struct promise_type {
    void return_value(int value) {
      ...
    }
    ...
  };
};

Result Coroutine() {
  ...
  co_return 1000;   // return_value 函数的参数 value 的值为 1000
}

// 也支持返回 void。只不过 promise_type 要定义的函数是 return_void 
struct Result {
  struct promise_type {
    void return_void() {
      ...
    }
    ...
  };
};

Result Coroutine() {
  ...
  co_return;
};

// 如果调用co_yield，对应的函数是yield_value
struct Result {
  struct promist_type {
    R yield_value(T v);
  };
};

Result Coroutine() {
  ...
  co_yield (T)v;  // co_await Result.yield_value(v);
}
```

#### 协程体抛出异常

```cpp
struct Result {
  struct promise_type {
    void unhandled_exception() {
      exception_ = std::current_exception(); // 获取当前异常
    }
    ...
  };
};
```

## `Awaitable` Interface

### Awaiters and Awaitables: Explaining operator `co_await`

- 一元运算符，仅可以在协程上下文内使用（本质上，包含co_await的任意方法，都会被编译为一个协程）(tautology: 同义反复)

- **支持`co_wait`运算符的类型被称为`Awaitable`**
  - `co_await` 操作符能否应用于某一类型取决于 `co_await` 表达式出现的上下文。`promise` 类型可以通过其 `await_transform` 方法改变例行程序 `co_await` 表达式的含义

* `Normally Awaitable`:  a type that supports the co_await operator in a coroutine context whose promise type does not have an await_transform member. 
* `Contextually Awaitable`: a type that only supports the co_await operator in the context of certain types of coroutines due to the presence of an await_transform method in the coroutine’s promise type.

- **实现了`await_ready`、`await_suspend`、 `await_resume`接口的类型被称为`Awaiter`**

- 一个类型可以同时是`Awaitable`和`Awaiter`

### Obtaining the Awaiter

**编译器首先做的就是为等待值（awaited value）生成包含`Awaiter`对象的代码。**，对于 `co_await <expr>` 表达式当中 `expr` 的处理，C++ 有一套完善的流程：

1. 如果 promise_type 当中定义了 await_transform 函数，那么先通过 `promise.await_transform(expr)` 来对 expr 做一次转换，得到的对象称为 awaitable；否则 awaitable 就是 expr 本身。
2. 接下来使用 awaitable 对象来获取等待体（awaiter）。如果 awaitable 对象有 operator co_await 运算符重载，那么等待体就是 `operator co_await(awaitable)`，否则等待体就是 awaitable 对象本身。

> Assume the promise object for the awaiting coroutine has type P
> And that promise is an l-value reference to the promise object for the current coroutine.
>
> If the promise type, P, has a member named await_transform then <expr> is first passed into a call to `promise.await_transform(<expr>)` to obtain the Awaitable value, awaitable. Otherwise, if the promise type does not have an await_transform member then we use **the result of evaluating <expr> directly** as the Awaitable object, awaitable.
>
> Then, if the Awaitable object, awaitable, has an applicable operator co_await() overload then this is called to obtain the Awaiter object. Otherwise the object, awaitable, is used as the awaiter object.

用伪代码表述如下：

```cpp
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
```

### Awaiting the Awaiter

assuming we have encapsulated the logic for turning the <expr> result into an Awaiter object into the above functions then the semantics of co_await <expr> can be translated (roughly) as follows:

```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready()) // 如果操作可能同步返回，不需要挂起，name await_ready 可以避免挂起的开销
  {
    using handle_t = std::experimental::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>   // 在 `<suspend-coroutine>` 的位置，编译器会生成一些代码来保存协程状态，以备未来恢复执行 (包括记录挂起点、寄存器数据)

    // 当前协程在 <suspend-coroutine> 操作完成后，就被认为已经被挂起了。挂起的协程可以恢复、销毁。

    if constexpr (std::is_void_v<await_suspend_result_t>)   // 返回 void 的 await_suspend 方法在返回时，无条件得将控制权转移给 caller/resumer
    {
      awaiter.await_suspend(handle_t::from_promise(p));   // 在 await_suspend 内部，是协程被挂起后可以被观察到的位置，await_suspend需要负责在未来某一时刻，将协程恢复或销毁
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p))) // 返回 bool 的 await_suspend 方法则在返回false时，则不经过控制权转移，直接在当前线程上恢复协程
      {
        <return-to-caller-or-resumer>   // 这里会将控制权转移回来，弹出local stack，保持coroutine frame存活
      }
    }

    <resume-point>  // 当协程最终被恢复的时候，执行点会到达这里
  }

  return awaiter.await_resume();  // auait_resume 的返回值，就是co_await表达式的返回值，这里也可能抛出异常
}
```

- 如果异常从 await_suspend 中抛出，name协程会立刻自动恢复，异常会传递出 co_await 表达式，跳过对 await_resume 的调用。

### Coroutine Handles

该类型表示协程框架的非拥有句柄，可用于恢复协程的执行或销毁协程框架。它还可用于访问协程的 Promise 对象。

```cpp
namespace std::experimental
{
  template<typename Promise>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void>
  {
    bool done() const;

    void resume();  // 在 resume-point 重新激活一个被挂起的协程。当协程下次到达 <return-to-caller-or-resumer> 点时，对 .resume() 的调用将返回。
    void destroy(); // 一般只有库作者在实现 Promise 的时候需要关注

    // 运行 coroutine handle 和 void *指针 之间互相转换
    void* address() const;
    static coroutine_handle from_address(void* address);
  };

  template<typename Promise>
  struct coroutine_handle : coroutine_handle<void>
  {
    Promise& promise() const;
    static coroutine_handle from_promise(Promise& promise); // 允许用协程Promise对象的引用来重建协程的控制器

    static coroutine_handle from_address(void* address);
  };
}
```

- Note that you must ensure that the type, P, exactly matches the concrete promise type used for the coroutine frame; attempting to construct a coroutine_handle<Base> when the concrete promise type is Derived can lead to undefined behaviour.

## Synchronisation-free async code

co_await 运算符的一个强有力的设计特性，就是允许再协程被挂起和恢复之间，执行其他逻辑

```shell
Time     Thread 1                           Thread 2
  |      --------                           --------
  |      ....                               Call OS - Wait for I/O event
  |      Call await_ready()                    |
  |      <supend-point>                        |
  |      Call await_suspend(handle)            |
  |        Store handle in operation           |
  V        Start AsyncFileRead ---+            V
                                  +----->   <AsyncFileRead Completion Event>
                                            Load coroutine_handle from operation
                                            Call handle.resume()
                                              <resume-point>
                                              Call to 8()
                                              execution continues....
           Call to AsyncFileRead returns
         Call to await_suspend() returns
         <return-to-caller/resumer>

```

- 将handle发布给其他线程之后，那么一个线程可能会在await_suspend()返回之前恢复另一个线程上的协程，并可能与await_suspend() 方法的其余部分同时执行。
- 协程恢复时要做的第一件事是调用await_resume()来获取结果，然后通常会立即销毁 Awaiter 对象（即await_suspend()调用的this指针）。然后，协程可能会运行完成，并在await_suspend() 返回之前销毁协程和 promise 对象。
- 因此，在await_suspend()方法中，一旦协程可以在另一个线程上同时恢复，您需要确保避免访问该对象或协程的.promise()对象，因为两者都可能已经被销毁。一般来说，在操作开始并且协程计划恢复后唯一可以安全访问的是 await_suspend() 中的局部变量。

### Comparison to Stackful Coroutines

### Avoiding memory allocations

- 当我们使用协程时，我们可以利用协程框架内的局部变量在协程挂起时保持活动状态的事实来避免为操作分配堆存储的需要。

### Example

实现一个基本的，awaitable的同步原语：一个异步的手动复位事件

- 是 awaitable 的，被多个并发执行的协程等待
- 当 awaited 时，需要挂起，等待线程调用 set 方法，此时所有等待的协程都会恢复
- 如果某个线程已经调用了 set 方法，则协程继续执行不挂起

理想情况下，还需要：noexcept、没有堆内存分片、lock-free

```cpp
// usage example
T value;
async_manual_reset_event event;

// A single call to produce a value
void producer()
{
  value = some_long_running_computation();

  // Publish the value by setting the event.
  event.set();
}

// Supports multiple concurrent consumers
task<> consumer()
{
  // Wait until the event is signalled by call to event.set()
  // in the producer() function.
  co_await event;

  // Now it's safe to consume 'value'
  // This is guaranteed to 'happen after' assignment to 'value'
  std::cout << value << std::endl;
}
```

```cpp
// interface
// two state, can be represented in std::atomic<void *>
class async_manual_reset_event
{
public:

  async_manual_reset_event(bool initiallySet = false) noexcept
    : m_state(initiallySet ? this : nullptr) {}

  // No copying/moving
  async_manual_reset_event(const async_manual_reset_event&) = delete;
  async_manual_reset_event(async_manual_reset_event&&) = delete;
  async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
  async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

  bool is_set() const noexcept { return m_state.load(std::memory_order_acquire) == this; }

  struct awaiter;
  awaiter operator co_await() const noexcept {
    return awaiter(*this);
  }

  void set() noexcept;
  void reset() noexcept {
    void* oldValue = this;
    m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
  }

private:

  friend struct awaiter;

  // - 'this' => set state
  // - otherwise => not set, head of linked list of awaiter*.
  mutable std::atomic<void*> m_state;

  // by storing the nodes within an ‘awaiter’ object that is placed within the coroutine frame will remove heap allocations
};

void async_manual_reset_event::set() noexcept
{
  // Needs to be 'release' so that subsequent 'co_await' has
  // visibility of our prior writes.
  // Needs to be 'acquire' so that we have visibility of prior
  // writes by awaiting coroutines.
  void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
  if (oldValue != this)
  {
    // Wasn't already in 'set' state.
    // Treat old value as head of a linked-list of waiters
    // which we have now acquired and need to resume.
    auto* waiters = static_cast<awaiter*>(oldValue);
    while (waiters != nullptr)
    {
      // Read m_next before resuming the coroutine as resuming
      // the coroutine will likely destroy the awaiter object.
      auto* next = waiters->m_next;
      waiters->m_awaitingCoroutine.resume();
      waiters = next;
    }
  }
}
```

```cpp
// define the awaiter
struct async_manual_reset_event::awaiter
{
  awaiter(const async_manual_reset_event& event) noexcept
  : m_event(event)
  {}

  bool await_ready() const noexcept;

  // most of the magic happens in an awaitable type
  bool await_suspend(std::experimental::coroutine_handle<> awaitingCoroutine) noexcept;
  
  void await_resume() noexcept {}

private:

  const async_manual_reset_event& m_event;
  std::experimental::coroutine_handle<> m_awaitingCoroutine;
  awaiter* m_next;
};

bool async_manual_reset_event::awaiter::await_ready() const noexcept {
  return m_event.is_set();
}

// - 保存协程句柄，等待后续调用 resume
// - 将awaiter原子入队，检查set状态，返回true或者false
bool async_manual_reset_event::awaiter::await_suspend(
  std::experimental::coroutine_handle<> awaitingCoroutine) noexcept
{
  // Special m_state value that indicates the event is in the 'set' state.
  const void* const setState = &m_event;

  // Remember the handle of the awaiting coroutine.
  m_awaitingCoroutine = awaitingCoroutine;

  // Try to atomically push this awaiter onto the front of the list.
  void* oldValue = m_event.m_state.load(std::memory_order_acquire);
  do
  {
    // Resume immediately if already in 'set' state.
    if (oldValue == setState) return false; 

    // Update linked list to point at current head.
    m_next = static_cast<awaiter*>(oldValue);

    // Finally, try to swap the old list head, inserting this awaiter
    // as the new list head.
  } while (!m_event.m_state.compare_exchange_weak(
             oldValue,
             this,
             std::memory_order_release,
             std::memory_order_acquire));

  // Successfully enqueued. Remain suspended.
  return true;
}
```

## `Promise` Interface

The `Promise` object defines and controls the behaviour of the coroutine itself by implementing methods that are called at specific points during execution of the coroutine.

当我们书写一个协程方法时，其方法内包含协程的关键字，name这个函数会被编译器转换为以下的形式。

相比于普通的方法，协程方法在真正执行之前，会有一些特定步骤被调用

```cpp
{
  co_await promise.initial_suspend();
  try
  {
    <body-statements>
  }
  catch (...)
  {
    promise.unhandled_exception();
  }
FinalSuspend:
  co_await promise.final_suspend();
}
```

通常会有以下步骤

1. Allocate a coroutine frame using operator new (optional).
2. Copy any function parameters to the coroutine frame.
3. Call the constructor for the promise object of type, P.
4. Call the promise.get_return_object() method to obtain the result to return to the caller when the coroutine first suspends. Save the result as a local variable.
5. Call the promise.initial_suspend() method and co_await the result.
6. When the co_await promise.initial_suspend() expression resumes (either immediately or asynchronously), then the coroutine starts executing the coroutine body statements that you wrote.

如果执行到 co_return 语句：

1. Call promise.return_void() or promise.return_value(<expr>)
2. Destroy all variables with automatic storage duration in reverse order they were created.
3. Call promise.final_suspend() and co_await the result.

如果在执行过程中抛出了未处理的异常

1. Catch the exception and call promise.unhandled_exception() from within the catch-block.
2. Call promise.final_suspend() and co_await the result.

删除协程帧也会包含一些步骤：

1. Call the destructor of the promise object.
2. Call the destructors of the function parameter copies.
3. Call operator delete to free the memory used by the coroutine frame (optional)
4. Transfer execution back to the caller/resumer.

### 分配协程帧

- `Promise` 类型可以重载 `operator new`，否则编译器将调用全局的 `operator new` 来构造协程帧。需要注意：
  - 传递给 operator new 的大小并不是 sizeof(Promise)，而是编译器根据协程参数的大小数量、Promise大小、局部变量的大小和数量等自动计算出来的大小
  - 编译器在一些条件下可以省略对 operator new 的调用，直接在调用者的栈帧上为协程分配内存
    - 如果协程帧的生命周期严格的小于调用者的生命周期，并且
    - 编译器能够在调用点看到协程帧所需要的大小
  - 标准未定义什么场景下，对 operator new 的省略一定会发生，所以请保持代码的鲁棒性
  - 在不适合使用`exception`的场景下，Promise 也提供了另外的选择，如果定义了一个静态方法 `P::get_return_object_on_allocation_failure()`，name编译器会重载一个 `operator new(size_t, nothrow_t)`，并在该调用返回nullptr的时候立刻调用静态方法，并将结果返回给调用者，而不是直接抛出一个异常

#### Customising coroutine frame memory allocaiton

```cpp
struct my_promise_type {
  void * operator new(std::size_t size) {
    void *p = my_custom_allocate(size);
    if {!p} throw std::bad_alloc{};
    return p;
  }

  void operator delete(void *p, std::size_t size) {
    my_custom_free(p, size);
  }
};
```

For example, you can implement operator new so that it allocates extra space after the coroutine frame and use that space to stash a copy of the allocator that can be used to free the coroutine frame memory.

```cpp
template<typename ALLOCATOR>
struct my_promise_type
{
  template<typename... ARGS>
  void* operator new(std::size_t sz, std::allocator_arg_t, ALLOCATOR& allocator, ARGS&... args)
  {
    // Round up sz to next multiple of ALLOCATOR alignment
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    // Call onto allocator to allocate space for coroutine frame.
    void* ptr = allocator.allocate(allocatorOffset + sizeof(ALLOCATOR));

    // Take a copy of the allocator (assuming noexcept copy constructor here)
    new (((char*)ptr) + allocatorOffset) ALLOCATOR(allocator);

    return ptr;
  }

  void operator delete(void* ptr, std::size_t sz)
  {
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    ALLOCATOR& allocator = *reinterpret_cast<ALLOCATOR*>(
      ((char*)ptr) + allocatorOffset);

    // Move allocator to local variable first so it isn't freeing its
    // own memory from underneath itself.
    // Assuming allocator move-constructor is noexcept here.
    ALLOCATOR allocatorCopy = std::move(allocator);

    // But don't forget to destruct allocator object in coroutine frame
    allocator.~ALLOCATOR();

    // Finally, free the memory using the allocator.
    allocatorCopy.deallocate(ptr, allocatorOffset + sizeof(ALLOCATOR));
  }
}
```

即便我们定制了协程的内存分配函数，编译器仍然不保证一定会调用该内存分配函数

### 拷贝参数到协程帧

- 必须拷贝，以保持生命周期合法
  - passed by value --> copy into 
  - passed by reference(either lvalue or rvale) --> copy reference into ,not the value they point to 
- 对于有 trivial destructors 的对象，如果在 <return-to-caller-or-resumer> 之后不存在对该对象的引用，那么编译器可以省略对该对象的copy
- 在协程场景使用引用传参是比较危险的 ref [Coroutines and Reference Parameters](https://toby-allsopp.github.io/2017/04/22/coroutines-reference-params.html)
- 如果在copy/move任何参数的过程中抛出了异常，协程会终止，所有已构造好的对象会被释放，协程帧被释放，异常会抛出给调用者

### Constructing the promise object

- 所有参数被copy到协程帧后，协程会构造promise对象
- 编译器会检查promise的构造函数重载，如果存在使用copy的参数的，则优先调用，否则会调用默认构造函数
- 遇到异常则终止

### Obtaining the return object

- 协程操作promise对象的第一件事，是获取 return-object，promise.get_return_object()。返回对象是协程在第一次挂起或者执行结束时，返回给调用者的对象。
  - 提前获取的原因是：协程栈帧有可能在开始执行后的某个点，又当前线程或者其他线程销毁；因此滞后获取时不安全的

```cpp
// Pretend there's a compiler-generated structure called 'coroutine_frame'
// that holds all of the state needed for the coroutine. It's constructor
// takes a copy of parameters and default-constructs a promise object.
struct coroutine_frame { ... };

T some_coroutine(P param)
{
  auto* f = new coroutine_frame(std::forward<P>(param));

  auto returnObject = f->promise.get_return_object();

  // Start execution of the coroutine body by resuming it.
  // This call will return when the coroutine gets to the first
  // suspend-point or when the coroutine runs to completion.
  coroutine_handle<decltype(f->promise)>::from_promise(f->promise).resume();

  // Then the return object is returned to the caller.
  return returnObject;
}
```

### The initial-suspend point

- 协程在完成帧初始化、获取返回值之后，接下来执行的就是 `co_await promise.initial_suspend();`
  - 这里，promise的作者可以控制，协程是直接执行function body，还是先挂起
    -如果在这里协程被挂起，那么可以通过coroutine_handle的resume来恢复、destroy来销毁
- 注意该调用点没有try catch保护，意味着异常发生时，协程会被销毁，异常会抛出给调用者
- 需要确保 return-type 是否包含RAII语义来释放协程帧，防止 double-free
- 大部分的 initial_suspend 返回 std::experimental::suspend_always 或者 std::experimental::suspend_never，两者都是noexcept awaitable

### Returning to the 

- 当协程执行完、或者到 return-to-caller-or-resume 点时，return-object会被返回给协程的调用者
- return-object 和协程函数的返回值类型不一定一致，必要的情况下，编译器会执行一个隐式转换

### Returning from the coroutine using `co_return`

当执行到 co_return 时，调用会转变为promise的相关return调用，以及一个 goto FinalSuspend 

- co_return;
  -> promise.return_void();
- co_return <expr>;
 -> <expr>; promise.return_void(); if <expr> has type void
 -> promise.return_value(<expr>); if <expr> does not have type void

- `goto FinalSuspend` 会释放所有自动存储期的本地变量，释放顺序与构造顺序相反，然后调用 co_await promise.final_suspend()
- 方法未显式调用 co_return 而执行到尾部时，类似在尾部包含一个 `co_return;`,在这种情况下，如果promise类型未定义 return_void，则会导致UB
- 此处抛出的异常，会传播到 promise.unhandled_exception()

### Handling exceptions that propagate out of the coroutine body

- 如果异常传播到协程方法体外，则异常会被捕获，并且在catch块中，promise.unhandled_exception()会被调用
  - 对该方法的典型实现是调用 `std::current_exception()`, 获取异常的copy，之后在别的上下文中抛出
  - 也可以立即重新抛出异常，这会导致协程立即被销毁。这可能会在 coroutine_handle::resume() noexcept 的情况下导致问题，除非你对resume的调用者有完全的掌控

### The final-suspend point

- 当执行完用户定义的代码时，结果通过 return_void return_value unhandled_exception 返回时，所有的局部变量会被销毁，在将执行权限交还给调用者之前，还可以执行一些逻辑。
- `co_await promise.final_suspend()`
  - 这里允许 publishing reautl，signalling completion、resuming a continuation
  - 或者立即挂起以防止协程执行完，协程帧被销毁
    - resume一个在该处挂起的协程是UB的，这里仅可以执行destroy

Note that while it is allowed to have a coroutine not suspend at the final_suspend point, it is recommended that you structure your coroutines so that they do suspend at final_suspend where possible. This is because this forces you to call .destroy() on the coroutine from outside of the coroutine (typically from some RAII object destructor) and this makes it much easier for the compiler to determine when the scope of the lifetime of the coroutine-frame is nested inside the caller. This in turn makes it much more likely that the compiler can elide the memory allocation of the coroutine frame.

### How the compiler chooses the promise type

- 编译器选择promise类型的方式是根据协程签名来进行类型萃取，std::experimental::coroutine_traits

如果有一个协程的函数签名如下

```cpp
task<float> foo(std::string x, bool flag);
```

那么编译器会将函数返回值和入参传递给萃取器，来推导promise的类型

```cpp
typename coroutine_traits<task<float>, std::string, bool>::promise_type;
```

如果协程函数是非静态成员函数，那么类类型会作为第二个模板参数传递进来。如果协程函数是一个右值引用的重载，那么模板参数也会是一个右值引用

```cpp
task<void> my_class::method1(int x) const;
task<foo> my_class::method2() &&;

// method1 promise type
typename coroutine_traits<task<void>, const my_class&, int>::promise_type;

// method2 promise type
typename coroutine_traits<task<foo>, my_class&&>::promise_type;
```

coroutine_traits 的默认实现是嵌套获取用户定义的 promise_type

```cpp
namespace std::experimental
{
  template<typename RET, typename... ARGS>
  struct coroutine_traits<RET, ARGS...>
  {
    using promise_type = typename RET::promise_type;
  };
}
```

如果你无法修改协程函数的返回值来定义其promise_type，那么你也可以重载一个coroutine_traits实现

```cpp
namespace std::experimental
{
  template<typename T, typename... ARGS>
  struct coroutine_traits<std::optional<T>, ARGS...>
  {
    using promise_type = optional_promise<T>;
  };
}
```

### Identifying a specific cotoutine activation frame

当一个协程方法被调用的时候，一个协程帧会被创建。那么如何保存对这个协程帧的引用呢？提案给出的方式是通过 `coroutine_handle` 类型，其接口定义可以类比如下：

```cpp
namespace std::experimental
{
  template<typename Promise = void>
  struct coroutine_handle;

  // Type-erased coroutine handle. Can refer to any kind of coroutine.
  // Doesn't allow access to the promise object.
  template<>
  struct coroutine_handle<void>
  {
    // Constructs to the null handle.
    constexpr coroutine_handle();

    // Convert to/from a void* for passing into C-style interop functions.
    constexpr void* address() const noexcept;
    static constexpr coroutine_handle from_address(void* addr);

    // Query if the handle is non-null.
    constexpr explicit operator bool() const noexcept;

    // Query if the coroutine is suspended at the final_suspend point.
    // Undefined behaviour if coroutine is not currently suspended.
    bool done() const;

    // Resume/Destroy the suspended coroutine
    void resume();
    void destroy();
  };

  // Coroutine handle for coroutines with a known promise type.
  // Template argument must exactly match coroutine's promise type.
  template<typename Promise>
  struct coroutine_handle : coroutine_handle<>
  {
    using coroutine_handle<>::coroutine_handle;

    static constexpr coroutine_handle from_address(void* addr);

    // Access to the coroutine's promise object.
    Promise& promise() const;

    // You can reconstruct the coroutine handle from the promise object.
    static coroutine_handle from_promise(Promise& promise);
  };
}
```

- 可以通过以下两种方式获取handle
  - 在`co_await`表达式中作为参数被传递给 `await_suspend()`
  - 若持有`promise`，则可以重新构造出handle `coroutine_hande<Promise>::from_promise()`
- coroutine_handle 不是一个 RAII 类型

### 自定义 co_await 的行为

通过定义 `Promise::await_transform()` 方法，编译器会自动将每个 `co_await <expr>` 转换为 `co_await promise.await_transform(<expr>)` 。

- lets you enable awaiting types that would not normally be awaitable.
- lets you disallow awaiting on certain types by declaring await_transform overloads as deleted.
- lets you adapt and change the behaviour of normally awaitable values

### 自定义 co_yield 的行为

If the co_yield keyword appears in a coroutine then the compiler translates the expression co_yield <expr> into the expression co_await promise.yield_value(<expr>). 

## Understanding Symmetric Transfer

> “symmetric transfer” which allows you to suspend one coroutine and resume another coroutine without consuming any additional stack-space.

### how a task coroutine works

```cpp
task foo() {
  co_return;
}

task bar() {
  co_await foo();
}
```

Let’s unpack what’s happening here when `bar()` evaluates `co_await foo()`:

- The bar() coroutine calls the foo() function. Note that from the caller’s perspective a coroutine is just an ordinary function.
- The invocation of foo() performs a few steps:
  - Allocates storage for a coroutine frame (typically on the heap)
  - Copies parameters into the coroutine frame (in this case there are no parameters so this is a no-op).
  - Constructs the promise object in the coroutine frame
  - Calls promise.get_return_object() to get the return-value for foo(). This produces the task object that will be returned, initialising it with a std::coroutine_handle that refers to the coroutine frame that was just created.
  - Suspends execution of the coroutine at the initial-suspend point (ie. the open curly brace)
  - Returns the task object back to bar().
- Next the bar() coroutine evaluates the co_await expression on the task returned from foo().
  - The bar() coroutine suspends and then calls the await_suspend() method on the returned task, passing it the std::coroutine_handle that refers to bar()’s coroutine frame.
  - The await_suspend() method then stores bar()’s std::coroutine_handle in foo()’s promise object and then resumes the foo() coroutine by calling .resume() on foo()’s std::coroutine_handle.
- The foo() coroutine executes and runs to completion synchronously.
- The foo() coroutine suspends at the final-suspend point (ie. the closing curly brace) and then resumes the coroutine identified by the std::coroutine_handle that was stored in its promise object before it was started. ie. bar()’s coroutine.
- The bar() coroutine resumes and continues executing and eventually reaches the end of the statement containing the co_await expression at which point it calls the destructor of the temporary task object returned from foo().
- The task destructor then calls the .destroy() method on foo()’s coroutine handle which then destroys the coroutine frame along with the promise object and copies of any arguments.

```cpp
class task {
  public:
    class promise_type {
      public:
        task get_return_object() noexcept {
          return task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        //  we want the coroutine to initially suspend at the open curly brace so that we can later resume the coroutine from this point when the returned task is awaited.
        // 1. 可以提前获取到对应的 coroutine_handle，防止由此带来的潜在竞争和同步开销
        // 2. 可以放心的由 task 的析构释放协程帧，不需要担心协程是否由其他线程已经执行完成而造成二次释放。同时利于编译器优化
        // 3. 增加了异常安全，如果调用者没有选择立刻 co_await returned_task, 而是执行了一些其他逻辑并出现了异常，这时可以放心地释放协程帧（因为我们知道其尚未执行），省去了很多麻烦的考虑（detaching、dangling reference、block in destructor、terminate、UB）
        std::suspend_always initial_suspend() noexcept { return {}; }

        // This method doesn’t actually need to do anything, it just needs to exist so that the compiler knows that co_return; is valid within this coroutine type.
        void return_void() noexcept {}

        // called if an exception escapes the body of the coroutine.
        void unhandled_exception() noexcept {
          std::terminate();
        }

        // when the coroutine execution reaches the closing curly brace, we want the coroutine to suspend at the final-suspend point and then resume its continuation. ie. the coroutine that is awaiting the completion of this coroutine.
        // 1. 将调用者恢复执行的动作延迟到当前协程挂起之后，是因为调用者可能会立即调用task.destructor释放协程帧，但对协程帧的释放，只有在协程挂起的状态下才是合法的，否则可能导致UB
        // 2. It’s important to note that the coroutine is not yet in a suspended state when the final_suspend() method is invoked. We need to wait until the await_suspend() method on the returned awaitable is called before the coroutine is suspended.
        struct final_awaiter {
          bool await_ready() noexcept { return false; }

          void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
            h.promise().continuation.resume();
          }

          void await_resume() noexcept {}
        };

        final_awaiter final_suspend() noexcept { return {}; }

        std::coroutine_handle<> continuation;
    };

    task(task &&t) noexcept : coro_(std::exchange(t,coro, {})) {}

    ~task() {   // RAII ensure
      if (coro_)
        coro_.destroy();
    }

    class awaiter {
      public:
        bool await_ready() noexcept { return false; }

        void await_suspend(std::coroutine_handle<> continuation) noexcept {
          // Store the continuation in the task's promise so that the final_suspend()
          // knows to resume this coroutine when the task completes.
          coro_.promise().continuation = continuation;
          // Then we resume the task's coroutine, which is currently suspended
          // at the initial-suspend-point (ie. at the open curly brace).
          coro_.resume();
        }

        void await_resume() noexcept {}

      private:
        explicit awaiter(std::coroutine_handle<task::promise_type> h) noexcept : coro_(h) {}

        std::coroutine_handle<task::promise_type> coro_;
    };

    awaiter operator co_await() && noexcept { return awaiter{coro_}; }

  private:
    explicit task(std::coroutine_handle<promise_type> h) noexcept : coro_(h) {}

    std::coroutine_handle<promise_type> coro_;
};
```

### TODO 一大串没看懂 。。。。。。