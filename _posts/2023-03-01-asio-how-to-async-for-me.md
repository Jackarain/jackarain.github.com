---
layout: post
title: asio异步化的实现方法
---

我们有时需要自己把一些阻塞操作在asio中去调用，如何将阻塞和asio的异步结合，又get_associated_executor在什么时候可能用到，这里直接上代码：

```c++
template<class Handler>
BOOST_ASIO_INITFN_RESULT_TYPE(Handler, void(boost::system::error_code, result_type))
async_do_something(Handler&& handler)
{
 // ASYNC_HANDLER_TYPE_CHECK(Handler, void(boost::system::error_code, result_type));
 boost::asio::async_completion<Handler, void(boost::system::error_code, result_type)> init(handler);

 std::thread(boost::launch::async, [handler = init.completion_handler]() mutable
 {
  boost::system::error_code ec;
  result_type result;

  // 执行操作，并拿到result或ec...
  // ...

  auto executor = boost::asio::get_associated_executor(handler);
  boost::asio::dispatch(executor, [ec, handler, result]() mutable
  {
    handler(ec, result);
  });
 }).detach();

 return init.result.get();
}
```

在这个代码中

```c++
BOOST_ASIO_INITFN_RESULT_TYPE(Handler, void(boost::system::error_code, result_type))
```

第2个参数 `void(boost::system::error_code, result_type)`，第一个是 `error_code`，第二个是 `async_do_something` 推导的返回类型，如果`handler` 是 `callback`，则会推导 `async_dosomething`返回为 `void`，如果是 `boost::asio::yield_context`，则会推导为 `result_type` 类型。

最终要做的事，是在 `std::thread` 中完成的，也就是说，真正执行的操作是在独立的线程中完成的。（当然这里是为了说明阻塞工作执行在另一个线程中，使用了线程来说明，也可以是其它异步操作并非需要开线程，但流程是一样）

当这个线程执行完真正的操作后

```c++
auto executor = boost::asio::get_associated_executor(handler);
```

这是拿到 `handler` 关联的执行器上下文，再通过 `boost::asio::dispatch` 投递到 `executor` 上执行 `handler`，实现线程切回调用者线程，这在协程中非常有必要。

例如, 我们最终使用这段代码是这样的：

```c++
boost::system:error_code ec; // 假设线程id是1
auto result = async_do_something(yield[ec]); // 那么假设在这个内部boost::async去执行实际操作的线程是2，async_do_something执行完成返回时，又将切回线程id是1
std::cout << result; // 输出async_do_something返回的结果，执行到这里就是在线程1上执行的了
```

如果没有 `auto executor = boost::asio::get_associated_executor(handler);` 我们不能保证线程切回 1

这篇文章大致讲了 `boost::asio::get_associated_executor` 的用途，也算是上篇文章的补充吧，实际上，如果考虑内存分配一至性，同样有`associated_allocator` 用于 `asio` 内部保存 `handler` 时内存分配器的统一定制，这里不作展开。

上面 `async_do_something` 的实现不兼容 `c++20` 协程方案，若要支持的话，代码需要变化一下，稍为麻烦一点，具体做法是：

```c++
struct initiate_do_something
{
 template <typename Handler>
 void operator()(Handler&& handler) const
 {
  boost::asio::detail::non_const_lvalue<Handler> handler2(handler);

  boost::async(boost::launch::async, [handler = std::move(handler2.value)]() mutable
  {
   boost::system::error_code ec;
   int result = 0x47;

   // 执行真正操作，并拿到result或ec...
   // ...

   auto executor = boost::asio::get_associated_executor(handler);
   boost::asio::dispatch(executor, [ec, handler = std::move(handler), result]() mutable
   {
    handler(ec, result);
   });
  });
 }
};

template<class Handler>
BOOST_ASIO_INITFN_RESULT_TYPE(Handler, void(boost::system::error_code, int))
async_do_something(Handler&& handler)
{
 return boost::asio::async_initiate<Handler,
  void(boost::system::error_code, int)>(initiate_do_something(), handler);
}
```

这样，我们就可以像asio其它方法一样，使用 `c++20` 协程方案了，如：

```c++
awaitable<void> do_something()
{
 int ret = co_await async_do_something(use_awaitable);
 std::cout << ret << std::endl;
}

// 在某处启动协程
co_spawn(ioc, do_something, detached);
```
