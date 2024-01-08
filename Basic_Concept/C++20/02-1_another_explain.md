# [Another Explain of coroutine](https://theshoemaker.de/posts/yet-another-cpp-coroutine-tutorial)

- If a function contains co_await, co_return or co_yield, it’s a coroutine.

- Coroutines are rewritten by the compiler in a certain way

```cpp
ReturnType someCoroutine(Parameters parameter)
{
    auto* frame = new coroutineFrame(std::forward<Parameters>(parameters));
    auto returnObject = frame->promise.get_return_object();
    co_await frame->promise.initial_suspend();
    try
    {
        <body-statements>
    }
    catch (...)
    {
        frame->promise.unhandled_exception();
    }
    co_await frame->promise.final_suspend();
    delete frame;
    return returnObject;
}
```

```cpp
class [[nodiscard]] Task {
public:
    struct FinalAwaiter {
        bool await_ready() const noexcept { return false; }

        template <typename P>
        auto await_suspend(std::coroutine_handle<P> handle) noexcept
        {
            return handle.promise().continuation;
        }

        void await_resume() const noexcept { }
    };

    struct Promise {
        std::coroutine_handle<> continuation;

        Task get_return_object()
        {
            return Task { std::coroutine_handle<Promise>::from_promise(*this) };
        }

        void unhandled_exception() noexcept { }

        void return_void() noexcept { }

        std::suspend_always initial_suspend() noexcept { return {}; }
        FinalAwaiter final_suspend() noexcept { return {}; }
    };
    using promise_type = Promise;

    Task() = default;

    ~Task()
    {
        if (handle_) {
            handle_.destroy();
        }
    }

    struct Awaiter {
        std::coroutine_handle<Promise> handle;

        bool await_ready() const noexcept { return !handle || handle.done(); }

        auto await_suspend(std::coroutine_handle<> calling) noexcept
        {
            handle.promise().continuation = calling;
            return handle;
        }

        void await_resume() const noexcept { }
    };

    auto operator co_await() noexcept
    {
        return Awaiter { handle_ };
    }

private:
    explicit Task(std::coroutine_handle<Promise> handle)
        : handle_(handle)
    {
    }

    std::coroutine_handle<Promise> handle_;
};
```