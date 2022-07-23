# [Coroutines](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

协程提案并没有明确规定协程的语义（semantics），而是定义了一个通用的机制，使得库代码可以通过实现符合特定接口的类型，来定制化协程的行为。

提案定义了两类主要的接口：`Promise` `Awaitable`

`Promise`接口定义了定制协程自身行为的方法

`Awaitable`接口定义了控制`co_await`语义的方法

## Awaiters and Awaitables: Explaining operator `co_await`

支持`co_wait`运算符的类型被称为`Awaitable`

* `Normally Awaitable`:  a type that supports the co_await operator in a coroutine context whose promise type does not have an await_transform member. 
* `Contextually Awaitable`: a type that only supports the co_await operator in the context of certain types of coroutines due to the presence of an await_transform method in the coroutine’s promise type.

实现了`await_ready`、`await_suspend`、 `await_resume`接口的类型被称为`Awaiter`

一个类型可以同时是`Awaitable`和`Awaiter`

### Obtaining the Awaiter

编译器首先做的就是为等待值（awaited value）生成包含`Awaiter`对象的代码。

假设：
* the promise object for the awaiting coroutine has type, P
*  that promise is an l-value reference to the promise object for the current coroutine.

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