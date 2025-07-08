---
layout: post
title: 理解 asio c++20 协程设计
---

`asio` 的 `c++20` 协程的设计十分复杂，并且和目前大多数的协程库有着非常不同的设计思路，通常大多数 `c++20` 协程的设计都是通过协程本身提供的机制构建一个协程调用链机制。

通常来说，要实现一个 `c++20` 协程就是通过协程的 `promise` 对象存储 `caller` 协程(调用者)的句柄(或干脆将整个调用者协程的 `promise` 对象指针一股脑存储在当前被调用的子协程的 `promise` 中)。

然后在子协程退出时(会触发调用 `final_suspend`), 在 `final_suspend()` 中调用 `caller` 协程的句柄然后 `resume` 让这个父协程得以继续往下执行。

通常来说，基于上述流程来实现协程库并不算太复杂，而 `asio c++20` 协程在设计上却大为不同，它完全通过一个 `pump` 的机制来驱动协程的运行，要实现 `pump` 这么一个机制，`asio` 于是设计了一套类似 `thread` 的协程结构，称之为 `awaitable_thread`，文档中的设计架构如下：

``` txt
                +------------------------------------+
                | top_of_stack_                      |
                |                                    V
 +--------------+---+                            +-----------------+
 |                  |                            |                 |
 | awaitable_thread |<---------------------------+ awaitable_frame |
 |                  |           attached_thread_ |                 |
 +--------------+---+           (Set only when   +---+-------------+
                |               frames are being     |
                |               actively pumped      | caller_
                |               by a thread, and     |
                |               then only for        V
                |               the top frame.)  +-----------------+
                |                                |                 |
                |                                | awaitable_frame |
                |                                |                 |
                |                                +---+-------------+
                |                                    |
                |                                    | caller_
                |                                    :
                |                                    :
                |                                    |
                |                                    V
                |                                +-----------------+
                | bottom_of_stack_               |                 |
                +------------------------------->| awaitable_frame |
                                                 |                 |
                                                 +-----------------+
```

在这个设计构架图中，`awaitable_frame` 则是协程调用链上的一个个 `Promise` 对象指针(这一点很重要)，`bottom_of_stack_` 始终指向协程入口 `awaitable_frame`，其中 `awaitable_frame` 之间通过 `caller_` 建立起链表关系，代码关系示例如：

``` c++
class awaitable_frame_base {
 awaitable_frame* caller_ = nullptr;
 awaitable_thread* attached_thread_ = nullptr;
};

class awaitable_frame : awaitable_frame_base {
 awaitable_frame_base* top_of_stack_;
};
```

有了这个调用链关系，实际上已经描述了整个设计的轮廓，但还远远不够，如 `attached_thread_` 它在这个结构中，始终处于调用栈顶的那个 `awaitable_frame` 当中，其它所有 `awaitable_frame` 都不得存储 `attached_thread_`，也就是虽然每个 `awaitable_frame` 都有成员 `attached_thread_`，但只有 `top_of_stack_` 指向的那个 `awaitable_frame` 才有资格存储 `attached_thread_`，这个 `attached_thread_` 其实也不是什么特别的东西，它主要就是存储整个协程入口点的 `awaitable` 对象，成员变量名为 `bottom_of_stack_`。

然后我们再来看 `class awaitable` 这个类型的一些关键点，它是一个标准的 `awaiter` 定义，下面代码省去一些无关的细节，主要抽出主体来分析

``` c++
class awaitable {
public:
  bool await_ready() const noexcept
  {
    return false;
  }

  template <class U>
  void await_suspend(std::coroutine_handle<> h)
  {
    frame_->push_frame(&h.promise());
  }

  T await_resume()
  {
    return frame_->get();
  }

private:
  awaitable_frame* frame_;
};

class awaitable_thread {
public:
  void launch()
  {
    bottom_of_stack_.frame_->top_of_stack_->attach_thread(this);
    pump();
  }

  void pump()
  {
    do
      bottom_of_stack_.frame_->top_of_stack_->resume();
    while (bottom_of_stack_.frame_ && bottom_of_stack_.frame_->top_of_stack_);
  }

private:
  // 协程入口 awaitable 对象
  awaitable<awaitable_thread_entry_point> bottom_of_stack_;
};
```

所以，可以看到，我们可以通过 `attached_thread_` 对象指针得到 `bottom_of_stack_`，再通过 `bottom_of_stack_` 可以得到入口协程的 `awaitable_frame` 信息，同时在这个 `awaitable_frame` 信息中也可得到 `top_of_stack_` 信息，而 `top_of_stack_.frame_` 本身的 `caller_` 指针就只能是空了(若是顶层协程)。

有了上面信息，再来看一个 `asio` 协程的异步具体是如何驱动的，这是最关键的地方，若有 `asio` 协程代码如下:

``` c++
net::awaitable<void> start_market_server()
{
    auto executor = co_await net::this_coro::executor;
    net::steady_timer timer(executor);
    timer.expires_after(1s);

    co_await timer.async_wait(net::use_awaitable);
}
```

这里重点分析 `co_await timer.async_wait(net::use_awaitable);` 这行代码的执行流程。

当然, 这是由 `use_awaitable` 和 `async_result` 类型通过 `async_initiate` 函数中使用 `async_result` 偏特化到指定的类型中(`async_result` 机制是 `asio` 定制 `CompletionToken` 的核心机制，这里不详谈)，这里会匹配到一个如下定义的 `async_result` 类型，其中还有一个 `static` 的协程函数，代码示例如下(为了便于分析，同样去掉了很多无关信息)。

```c++
class async_result<use_awaitable_t>
{
  typedef typename awaitable_handler<> handler_type;
  typedef typename handler_type::awaitable_type return_type;

  static handler_type* do_init(
      auto* frame, Initiation& initiation,
      use_awaitable_t u, InitArgs&... args)
  {
    (void)u;

    handler_type handler(frame->detach_thread());
    std::move(initiation)(std::move(handler), std::move(args)...);

    return nullptr;
  }

  template <typename Initiation, typename... InitArgs>
  static return_type initiate(Initiation initiation,
    use_awaitable_t u, InitArgs... args)
  {
    co_await [&] (auto* frame)
    {
        return do_init(frame, initiation, u, args...);
    };

    for (;;) {} // Never reached.
  }
};
```

重点在这个 `initiate` 协程中, 这个协程会根据 `return_type` (推导后实际上也是 `awaitable` 类型) 构造一个 `frame`，也就是 `initiate` 协程的帧，这个 `frame` 并存储在返回类型的 `awaitable` 类型的临时对象中, 然后由用户代码中的 `co_await` 调用起这个 `initiate` 协程。

当执行 `co_await` 在这个 `initiate` 协程上，就会在新构造的 `awaitable` 的 `await_suspend` 中执行 `frame->push_frame` 将 `caller` 保存到当前 `frame` 中的 `caller_` 成员中，同时将 `attached_thread_` 这个指针从 `caller` 中转移到当前 `frame` 中，同时还修改 `entry_point` 的 `top_of_stack_` 为当前 `frame` (也就是 `initiate` 协程的帧)，需要注意的是 `entry_point` 就是 `bottom_of_stack_.frame_`, 也就是入口点协程，`top_of_stack_` 就是在入口点协程 `frame` 上的，即是：`bottom_of_stack_.frame_->top_of_stack_`。

当上述流程执行完后，会返回到 `caller` 的 `pump()` 中循环当中，也就是 `top_of_stack_->resume();`，紧接着执行循环条件判断

```c++
do
  bottom_of_stack_.frame_->top_of_stack_->resume();
while (bottom_of_stack_.frame_ && bottom_of_stack_.frame_->top_of_stack_);
```

由于在前面保存了 `top_of_stack_` 为当前 `frame` (即：`async_result::initiate` 协程)，所以条件继续为 `true`, 继续循环，执行当前 `initiate` 协程帧的 `frame->resume()`，从而开始运行 `initiate` 这个协程函数。

在这个 `initiate` 协程函数中，`co_await [&] (auto* frame)` 将会被调用触发 `await_transform(Function f)` 调用，在这里面的 `await_suspend` 里设置 `frame->resume_context_` 这个临时变量 `resume_context` 指针，以便在 `resume()` 调用完成后可执行 `[&] (auto* frame)` 这个 `lambda` 函数，在这个 `lambda` 中, 参数 `frame` 指针是当前 `initiate` 协程的帧。

在 `initiate` 协程中，由于只 `resume()` 了一次，因此只会执行一次 `co_await`，后面的代码则不再会有机会得到执行

```c++
  template <typename Initiation, typename... InitArgs>
  static return_type initiate(Initiation initiation,
    use_awaitable_t u, InitArgs... args)
  {
    co_await [&] (auto* frame)
    {
        return do_init(frame, initiation, u, args...);
    };

    for (;;) {} // 永远不会执行到的地方
  }
```

这就是这个代码的的奇怪之处，再回到上面 `lambda` 函数的执行流程中，`do_init` 中将当前 `frame` 的 `attached_thread_` 转移到 `awaitable_handler` 中（`awaitable_handler` 就是 `attached_thread_` 相同基类 `awaitable_thread` 的类型）的叫 `handler` 的对象中，然后通过 `async_initiation` 的异步启动函数对象发起实际异步，这样 `awaitable_handler` 就被携带在发起的异步 `op` (通过 `operation` 类型擦除)中了，`initiate` 协程从而实现了和 `async_initiation` 中 `Initiation` 的桥接起来，使得用户实现 `async_initiation` 中的 `Initiation` 函数对象能在回调或协程等不同形式的 `CompletionToken` 下适配。

需要注意的是，在 `lambda` 中的代码:

```c++
handler_type handler(frame->detach_thread());
std::move(initiation)(std::move(handler), std::move(args)...);
```

第一行代码它会将 `awaitable_thread` 对象 `move` 复制到 `handler` 对象，在这个 `move` 操作中非常关键的是 `move` 构造函数

```c++
awaitable_thread(awaitable_thread&& other) noexcept
: bottom_of_stack_(std::move(other.bottom_of_stack_))
{
}
```

在这个 `move` 构造函数中，它会将之前的 `awaitable_thread` 对象中的 `bottom_of_stack_` 对象的 `frame` 通过 `move` 构造到当前 `awaitable_thread` 对象的 `bottom_of_stack_` 的 `frame`，在这里的 `awaitable_thread` 也就是 `awaitable_handler` 对象，这里也将导致原来的 `awaitable_thread` 对象的 `frame` 变为空指针，这一点十分重要。

第二行 `std::move(initiation)(std::move(handler), std::move(args)...);` 即是 `async_initiation` 中 `Initiation` 对象，正是通过此 `lambda` 桥接到它，在 `async_initiation` 中 `Initiation` 真正发起异步操作。

在 `lambda` 执行完成后，`await_transform(Function f)` 调用就会返回，同时会返回到 `initiate` 协程上次 `resume()` 的地方，也就是 `pump()` 这个函数的循环当中，然后再判断循环条件时

```c++
while (bottom_of_stack_.frame_ && bottom_of_stack_.frame_->top_of_stack_);
```

此时，因为之前在 `lambda` 中经历了 `move` 构造，`awaitable_thread::pump()` 函数的当前 `awaitable_thread` 的 `bottom_of_stack_.frame_` 就被转移到了 `awaitable_handler` 对象的 `bottom_of_stack_.frame_` 中了，所以此时这里条件不成立因此退出 `pump()` 函数，执行流程又再次返回到 asio 的 `io_context` 事件循环当中。

当异步操作完成，事件循环再次触发 `op` 的回调，这个 `op` 通过类型恢复后，它就是前面说的 `awaitable_handler` 对象，事件循环 `run` 将调用 `awaitable_handler` 对象的 `operator()` 函数，下面拿其中一个版本来尝试说明

```c++
template <typename Arg0, typename Arg1>
void operator()(Arg0&& arg0, Arg1&& arg1)
{
    this->frame()->attach_thread(this);
    if constexpr (is_disposition<T0>::value)
    {
        if (arg0 == no_error)
            this->frame()->return_value(std::forward<Arg1>(arg1));
        else
            this->frame()->set_disposition(std::forward<Arg0>(arg0));
    }
    else
    {
        this->frame()->return_values(std::forward<Arg0>(arg0),
            std::forward<Arg1>(arg1));
    }
    this->frame()->clear_cancellation_slot();
    this->frame()->pop_frame();
    this->pump();
}
```

`awaitable_handler` 的 `operator()(Arg0&& arg0, Arg1&& arg1)` 函数，在这个函数当中，通过 `attach_thread` 到当前 `this` 也就是 `awaitable_thread` 指针(即 `awaitable_handler` 本身)，并将上面 `arg0, arg1` 设置到当前 `frame` 当中(注意，当前 `frame` 也就是之前 `initiate` 协程的 `frame` 指针), 紧接着调用 `pop_frame()` 将原来存储在 `frame` 当中的 `caller_` 作为 `top_of_stack_`，以及将 `attached_thread_` 指针转移到 `caller_` 中的 `attached_thread_` 指针上，并清空当前 `frame` 的 `caller_` 和 `attached_thread_` 成员，然后再执行 `pump()`，在 `pump()` 调用中，实际上调用的是 `caller_` 的 `resume()`, 从而将唤醒 `caller_` 协程，也就是用户代码中:

```c++
co_await timer.async_wait(net::use_awaitable);
```

此时，将会执行协程 `awaitable` 对象的 `await_resume()` 以获取返回值

```c++
T await_resume()
{
    return awaitable(static_cast<awaitable&&>(*this)).frame_->get();
}
```

这里关键的一点就是协程返回值是存储在 `initiate` 协程的 `frame_` 指针上(也就是它的 `promise` 类型中的 `result_`)，`await_resume()` 的返回值就是整个异步协程调用的返回值，这里是实现了协程返回值的关键。

当整个 `co_await` 语句包括返回值都执行完成后，`awaitable` 这个临时对象，就会进入析构，此时才最终删除 `initiate` 协程的 `promise` 上所有信息，同时也包括 `std::coroutine_handle` 指针(`coro_.destroy();`), 此时才总算圆满的将整个 `co_await` 一个 `asio` 协程流程执行完成。

通过上面的分析，可以看到 `asio` 协程的设计思路非常不一般，虽然十分复杂，却也非常精巧，而且并没有多余的开销实现，没有 `new`，没有锁，也没有系统调用，仅是一些必要的逻辑语句，因此十分高效。

由于 `asio` 协程的复杂度极高，所以这篇文章建议对着 `asio` 源码来阅读，这才能抽丝剥茧一般解惑 `asio` 协程的设计。
