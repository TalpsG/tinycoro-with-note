
之前花过几天看过c++的coroutine，看的教程倒不少，可以没看明白C++20的协程到底要怎么用？
最近有个大佬搞了一个`tinycorolab`，是一个类似`15445/6.824 lab`的一个项目，使用`liburing/coroutine`实现的异步io框架，有手册，有解答的代码。

我做了个开头，后面就直接读了这个`tinycoro`(算是lab的参考答案)的代码，结合lab手册加上之前看的一点`rust async`相关的东西，总算是了解了C++的协程应该怎么用。
下面会分几篇文章讲下这个库中比较重要的代码. 阅读的前提是要了解`c++ coroutine`的一些关键字，以及关键字的效果比如`co_await`会`awaiter`的什么函数？

本文要讲的代码在 `include/coro/task.hpp`,`src/task.cpp`

# c++ 协程
一个协程类似这样：
```cpp
Task<return_type> task_2(){
  // do something
}

Task<return_type> task_1(){
  // do something 

  co_await task_2();

  // do something
}
int main(){
  auto coro = task_1();
  while(coro.done()){
    coro.resume();
  }
  coro.destroy();
}
```
通过协程函数返回的协程对象来对协程进行管理，比如`resume/destroy`。协程可以注意是可以不是必须，在`co_await`处暂停，并且通过`coroutine_handle.resume`来返回到`co_await/co_yield`处继续执行。

所以可以把协程理解成一个在某个点可以`resume/suspend`的函数。我们可以把一些任务实现成一个协程函数，提交到后台线程，后台线程去调度执行已提交的全部协程。这种方式相比
线程池`std::async`的方式开销要小很多，因为协程的任务切换是在用户态的，且协程内部异步io时可切走到其他协程，`std::async`只能`yield`。

# promise
```cpp
struct promise_base
{
    friend struct final_awaitable;
    struct final_awaitable
    {
        constexpr auto await_ready() const noexcept -> bool { return false; }

        template<typename promise_type>
        auto await_suspend(std::coroutine_handle<promise_type> coroutine) noexcept -> std::coroutine_handle<>
        {
            // If there is a continuation call it, otherwise this is the end of the line.
            auto& promise = coroutine.promise();
            return promise.m_continuation != nullptr ? promise.m_continuation : std::noop_coroutine();
        }

        constexpr auto await_resume() noexcept -> void {}
    };

    promise_base() noexcept = default;
    ~promise_base()         = default;

    constexpr auto initial_suspend() noexcept { return std::suspend_always{}; }

    [[CORO_TEST_USED(lab1)]] constexpr auto final_suspend() noexcept { return final_awaitable{}; }

    auto continuation(std::coroutine_handle<> continuation) noexcept -> void { m_continuation = continuation; }

    inline auto set_state(coro_state state) -> void { m_state = state; }

    inline auto get_state() -> coro_state { return m_state; }

    inline auto is_detach() -> bool { return m_state == coro_state::detach; }

protected:
    std::coroutine_handle<> m_continuation{nullptr};
    coro_state              m_state{coro_state::normal};

};
```

协程类的内部需要有一个`promise`，`promise_base`是其基类，其把返回值类型无关的东西抽了出来，组织成了`promise_base`。
最关键的是`promise_base`中有一个`m_continuation`这个`handle`，这个`handle`保存的是父协程的`handle`，用来返回父协程中断处继续执行，就像是函数调用一样。
在协程结束后会`co_await`内部`promise_type::final_suspend`的返回对象`awaiter`，我们在`await_suspend`当中直接返回父协程的`handle`就可以了，这样会自动调用该`handle`的`resume`方法，切回到父协程。
后面看完`task`后会梳理一下整个流程。

# task
```cpp
template<typename return_type>
class [[CORO_AWAIT_HINT]] task
{
public:
    using task_type        = task<return_type>;
    using promise_type     = detail::promise<return_type>;
    using coroutine_handle = std::coroutine_handle<promise_type>;

    struct awaitable_base
    {
        awaitable_base(coroutine_handle coroutine) noexcept : m_coroutine(coroutine) {}

        auto await_ready() const noexcept -> bool { return !m_coroutine || m_coroutine.done(); }

        auto await_suspend(std::coroutine_handle<> awaiting_coroutine) noexcept -> std::coroutine_handle<>
        {
            m_coroutine.promise().continuation(awaiting_coroutine);
            return m_coroutine;
        }

        std::coroutine_handle<promise_type> m_coroutine{nullptr};
    };

    task() noexcept : m_coroutine(nullptr) {}

    explicit task(coroutine_handle handle) : m_coroutine(handle) {}
    task(const task&) = delete;
    task(task&& other) noexcept : m_coroutine(std::exchange(other.m_coroutine, nullptr)) {}

    ~task()
    {
        if (m_coroutine != nullptr)
        {
            m_coroutine.destroy();
        }
    }

    auto operator=(const task&) -> task& = delete;

    auto operator=(task&& other) noexcept -> task&
    {
        if (std::addressof(other) != this)
        {
            if (m_coroutine != nullptr)
            {
                m_coroutine.destroy();
            }

            m_coroutine = std::exchange(other.m_coroutine, nullptr);
        }

        return *this;
    }

    /**
     * @return True if the task is in its final suspend or if the task has been destroyed.
     */
    auto is_ready() const noexcept -> bool { return m_coroutine == nullptr || m_coroutine.done(); }

    auto resume() -> bool
    {
        if (!m_coroutine.done())
        {
            m_coroutine.resume();
        }
        return !m_coroutine.done();
    }

    auto destroy() -> bool
    {
        if (m_coroutine != nullptr)
        {
            m_coroutine.destroy();
            m_coroutine = nullptr;
            return true;
        }

        return false;
    }

    [[CORO_TEST_USED(lab1)]] auto detach() -> void
    {
        assert(m_coroutine != nullptr && "detach func expected no-nullptr coroutine_handler");
        auto& promise = m_coroutine.promise();
        promise.set_state(detail::coro_state::detach);
        m_coroutine = nullptr;
    }

    auto operator co_await() const& noexcept
    {
        struct awaitable : public awaitable_base
        {
            auto await_resume() -> decltype(auto) { return this->m_coroutine.promise().result(); }
        };

        return awaitable{m_coroutine};
    }

    auto operator co_await() const&& noexcept
    {
        struct awaitable : public awaitable_base
        {
            auto await_resume() -> decltype(auto) { return std::move(this->m_coroutine.promise()).result(); }
        };

        return awaitable{m_coroutine};
    }

    auto promise() & -> promise_type& { return m_coroutine.promise(); }
    auto promise() const& -> const promise_type& { return m_coroutine.promise(); }
    auto promise() && -> promise_type&& { return std::move(m_coroutine.promise()); }

    auto handle() & -> coroutine_handle { return m_coroutine; }
    auto handle() && -> coroutine_handle { return std::exchange(m_coroutine, nullptr); }

private:
    coroutine_handle m_coroutine{nullptr};
};
```

`task`中提供了`resume/destroy`用来对外提供接口来继续执行对应的协程或是销毁协程。

我们还是来看看嵌套协程调用的例子。

```cpp
Task<return_type> task_2(){
  // do something
}
Task<return_type> task_1(){
  // do something 

  co_await task_2();

  // do something
}
int main(){
  auto t = task_1();
  t.resume();
}
```

协程运行结束后会对`promise_type::final_suspend`的返回值`co_await`，我们的`Task`结束后会对`promise_base::final_suspend`返回值`co_await`，因此这里`promise_base::final_suspend`不能直接返回`suspend_always`，因为这样相当于直接退出协程将管理权给到了协程之外，也就是返回到`main`函数里了。这样是不对的，因为如果是子协程运行结束后需要跳转到父协程调用子协程的位置继续向下执行，就像函数调用一样。

子协程的退出调用是这样的
1. `co_return`或协程函数结束
2. `co_await promise_base::final_suspend()`，

如果我们想要返回父协程暂停的位置，就要在`final_suspend`返回的`awaiter`里做文章，我们需要在这个`awaiter`的`await_suspend`里获取到父协程的`handle`，直接返回即可，`await_suspend`返回类型是`handle`时会直接`resume`这个协程，也就是切到这个协程上。
我们可以看到前一节的`promise_base`的`final_awaitable`的`await_suspend`已经做了这件事了，`m_continuation`就是保存的父协程的`handle`
```cpp
struct final_awaitable
{
    constexpr auto await_ready() const noexcept -> bool { return false; }

    template<typename promise_type>
    auto await_suspend(std::coroutine_handle<promise_type> coroutine) noexcept -> std::coroutine_handle<>
    {
        // If there is a continuation call it, otherwise this is the end of the line.
        auto& promise = coroutine.promise();
        return promise.m_continuation != nullptr ? promise.m_continuation : std::noop_coroutine();
    }

    constexpr auto await_resume() noexcept -> void {}
};
```
问题来了，我们在哪里保存父协程的`handle`到`promise_base`? 这就是为什么我们要重载`co_await`(其实实现`await_ready/suspend/resume`这三个方法也行)。
先梳理一下嵌套调用的流程
1. `task_1`父协程`Task`对象构造
2. 父协程`do something`
3. `task_2`子协程`task`对象构造
4. `co_await`这个子协程对象
5. 子协程`do something`
6. 子协程退出，后面就是前面提到的子协程退出调用了。

在第4步的时候是要先构造`task`对象，然后再对`co_await task`的，而`co_await task`实际上是`co_await (task.operator co_await(awaiting_handle));`这样的。这里参数的`awaiting_handle`实际上就是`co_await`所在的协程，也就是父协程。既然知道父协程在哪了？我们也拿到了父协程的`handle`，只需要在`await_suspend`中将其保存到当前协程也就是子协程的`promise`当中即可。

`task`内特地实现了一个`awaitable_base`就是用来在`co_await`返回使用的，我们在重载函数内构造一个`awaiter`对象用来`co_await`，这个对象的`await_suspend`拿到了父协程的`handle`，在子协程`handle`的`promise`中保存了起来。这样在最后就可以利用这个`handle`切回父协程了。

```cpp
auto await_suspend(std::coroutine_handle<> awaiting_coroutine) noexcept -> std::coroutine_handle<>
{
    m_coroutine.promise().continuation(awaiting_coroutine);
    return m_coroutine;
}
```

写得有些乱，请谅解，下篇文章是`tinycoro`的`engine/context/scheduler`
