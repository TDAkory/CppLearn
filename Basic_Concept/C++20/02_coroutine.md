# [Coroutines](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

协程提案并没有明确规定协程的语义（semantics），而是定义了一个通用的机制，使得库代码可以通过实现符合特定接口的类型，来定制化协程的行为。

提案定义了两类主要的接口：`Promise` `Awaitable`

`Promise`接口定义了定制协程自身行为的方法

`Awaitable`接口定义了控制`co_await`语义的方法

## Awaiters and Awaitables: Explaining operator `co_await`