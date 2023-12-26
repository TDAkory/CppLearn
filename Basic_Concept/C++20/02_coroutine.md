# [Coroutines](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

- [Coroutines](#coroutines)
  - [Refs](#refs)
  - [What does the Coroutines TS give us?](#what-does-the-coroutines-ts-give-us)
  - [Explain in cppreference](#explain-in-cppreference)
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


## Refs

- [Coroutine Theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory)
- [C++ Coroutines: Understanding operator co_await](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

- [Coroutines](https://en.cppreference.com/w/cpp/language/coroutines)

## What does the Coroutines TS give us?

* Three new language keywords: co_await, co_yield and co_return
* Several new types in the std::experimental namespace:
  * `coroutine_handle<P>`
  * `coroutine_traits<Ts...>`
  * `suspend_always`
  * `suspend_never`
* A general mechanism that library writers can use to interact with coroutines and customise their behaviour.
* A language facility that makes writing asynchronous code a whole lot easier!

协程提案并没有明确规定协程的语义（semantics），而是定义了一个通用的机制，使得库代码可以通过实现符合特定接口的类型，来定制化协程的行为。

提案定义了两类主要的接口：`Promise` `Awaitable`

- `Promise`接口定义了定制协程自身行为的方法：库作者可以定制coroutine在 `called`、`return`、`co_await`\`co_yield` 时的行为

`Awaitable`接口定义了控制`co_await`语义的方法：当一个变量是 `co_await`ed，那么编译器会生成一系列可指定的针对 `awaitable` 变量的方法，这些方法包括：是否挂起协程、在挂起后执行某些逻辑、在恢复时执行某些逻辑来处理`co_await`的返回值。

## Explain in cppreference

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

编译器首先做的就是为等待值（awaited value）生成包含`Awaiter`对象的代码。

假设：

* the promise object for the awaiting coroutine has type P
* that promise is an l-value reference to the promise object for the current coroutine.

If the promise type, P, has a member named await_transform then <expr> is first passed into a call to `promise.await_transform(<expr>)` to obtain the Awaitable value, awaitable. Otherwise, if the promise type does not have an await_transform member then we use **the result of evaluating <expr> directly** as the Awaitable object, awaitable.

Then, if the Awaitable object, awaitable, has an applicable operator co_await() overload then this is called to obtain the Awaiter object. Otherwise the object, awaitable, is used as the awaiter object.

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
                                              Call to await_resume()
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