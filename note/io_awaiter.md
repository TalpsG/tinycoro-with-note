# base_io_awaiter
定义了`awaiter`必须的几个方法，在构造之初就获取了一个`sqe`用来后续的io操作。
```cpp
class base_io_awaiter
{
public:
    base_io_awaiter() noexcept : m_urs(coro::detail::local_engine().get_free_urs())
    {
        // TODO: you should no-block wait when get_free_urs return nullptr,
        // this means io submit rate is too high.
        assert(m_urs != nullptr && "io submit rate is too high");
    }

    constexpr auto await_ready() noexcept -> bool { return false; }

    auto await_suspend(std::coroutine_handle<> handle) noexcept -> void { m_info.handle = handle; }

    auto await_resume() noexcept -> int32_t { return m_info.result; }

protected:
    io_info             m_info;
    coro::uring::ursptr m_urs;
};
```

其中的`io_info`定义如下
```cpp
struct io_info
{
    coroutine_handle<> handle;
    int32_t            result;
    io_type            type;
    uintptr_t          data;
    cb_type            cb;
};
```
`iouring`允许设置一个`sqe`的`data`指针，用来存放用户自定义的数据。我们可以在`cqe`中拿到提交`sqe`前设置的值，这样就可以在io完成后做一些特定任务。
比如我们设置`data`为`&io_info`，在io完成后，`engine`会执行其`(io_info*)data->cb`来执行后续操作。
```cpp
auto engine::handle_cqe_entry(urcptr cqe) noexcept -> void
{
    auto data = reinterpret_cast<net::detail::io_info*>(io_uring_cqe_get_data(cqe));
    data->cb(data, cqe->res);
}
```

有些抽象，我们结合一个例子来说明`awaiter`的作用。
# tcp_accept_awaiter
假设我们有一个协程函数是这样的：
```cpp
Task func(int port){
  int flags = 0;
  co_await tcp_accept_awaiter(port, flags);

  // 后续 io 操作
}
```

这里有一个`tcp_accept_awaiter`，其主要作用就是用来监听某个端口，异步的`accept`连接请求。
```cpp

class tcp_accept_awaiter : public detail::base_io_awaiter
{
public:
    tcp_accept_awaiter(int listenfd, int flags) noexcept;

    static auto callback(io_info* data, int res) noexcept -> void;

private:
    inline static socklen_t len = sizeof(sockaddr_in);
};
```
其构造函数如下：
```cpp

tcp_accept_awaiter::tcp_accept_awaiter(int listenfd, int flags) noexcept
{
    m_info.type = io_type::tcp_accept;
    m_info.cb   = &tcp_accept_awaiter::callback;

    // FIXME: this isn't atomic, maybe cause bug?
    io_uring_prep_accept(m_urs, listenfd, nullptr, &len, flags);
    io_uring_sqe_set_data(m_urs, &m_info); // old uring version need set data after prep
    local_engine().add_io_submit();
}
```
c++的对象构造是先构造父类，所以构造函数体内 基类已经构造完毕了，拿到`sqe`了，此时设置`info`信息放入`sqe`提交io请求即可。
```cpp

auto tcp_accept_awaiter::callback(io_info* data, int res) noexcept -> void
{
    data->result = res;
    submit_to_context(data->handle);
}
```
`callback`非常简单，就是设置返回值到`info`然后提交`awaiter`所处的协程到`context`，后续我们的协程就会被`engine`调度执行。

## 这里为什么需要重新handle到context?
因为`engine`实现的时候，每次执行协程都会先将其`handle`从队列中`pop`出来，所以我们需要在`callback`中将其添加回去，否则该`handle`对应的协程就变成野的协程了。




# tcp echo server 
下面来看看`examples/tcp_echo_server.cpp`的代码。
```cpp
#include "coro/coro.hpp"

using namespace coro;

#define BUFFLEN 10240

task<> session(int fd)
{
    char buf[BUFFLEN] = {0};
    auto conn         = net::tcp_connector(fd);
    int  ret          = 0;
    while ((ret = co_await conn.read(buf, BUFFLEN)) > 0)
    {
        log::info("client {} receive data: {}", fd, buf);
        ret = co_await conn.write(buf, ret);
    }

    ret = co_await conn.close();
    log::info("client {} close connect", fd);
    assert(ret == 0);
}

task<> server(int port)
{
    log::info("server start in {}", port);
    auto server = net::tcp_server(port);
    int  client_fd;
    while ((client_fd = co_await server.accept()) > 0)
    {
        log::info("server receive new connect");
        submit_to_scheduler(session(client_fd));
    }
}

int main(int argc, char const* argv[])
{
    /* code */
    scheduler::init();

    submit_to_scheduler(server(8000));
    scheduler::loop();
    return 0;
}
```
可以看到`server`协程中先创建了`tcp_server`这个对象，该对象构造时创建了`socket`以及`bind`了对应`port`。下面就`co_await accept()`了，这个方法会返回一个`tcp_accept_awaiter`，最后返回`accept`后的`sockfd`。

然后在循环中提交`session`协程到`scheduler`，每一个`session`协程内会创建`tcp_connector`对象，然后异步读取数据，异步写入数据。这两个异步操作都是通过`co_await conn.read/write()`实现的。这两个`awaiter`都是在构造时设置`info`，提交io申请，在io完成后`engine`执行`callback`中将当前协程重新提交到`ctx`。


