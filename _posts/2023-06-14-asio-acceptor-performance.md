---
layout: post
title: 关于在 asio 使用 tcp::acceptor 时性能优化
---

在 `asio` 中实现大并发连接的服务器时，如 `hls` 服务器，客户端连接请求十分频繁且都是短连接为主，由其是成
千上万客户端连接请求的时候，此时非常需要注意一个问题，那就是在使用 `tcp::acceptor` 时，很可能会导致客户
端连接上的瓶颈。我们通常会写出下面这样一段代码用来做接受客户端连接：

```c++
asio::awaitable<void> start_server_listen(tcp::acceptor& acceptor)
{
  boost::system::error_code error;
  auto executor = co_await asio::this_coro::executor;

  while (true) {
    tcp::socket socket(executor);

    co_await acceptor.async_accept(
      socket, net::use_awaitable[error]);

    if (error) {
      // error 处理...
      break;
    } else {
      // 一些必要的业务流程.
      // ...
      // 创建一个 client_proc 协程处理 socket 上的数据传输.
      asio::co_spawn(
        executor,
        client_proc(std::move(socket)),
        asio::detached);
    }
  }

  co_return;
}
```
这个代码对于一般服务器完全足够应付，但是在大并发，客户端高频率的短连接中，就会出现效率问题，分析原因如下：
在 `async_accept` 得到响应时，正在处理 `co_await` 下面的其它业务流程，也包括创建新协程 `client_proc`，
而在与此同时，`acceptor` 将不会响应任何客户端连接请求，因为它此时并没有在 `async_accept` 中。

对于上面这个问题，解决办法其实也很简单，那就是同时发起多个 `async_accept` 即可，代码如下：
```c++
asio::awaitable<void> start_server_listen(tcp::acceptor& acceptor)
{
  auto executor = co_await asio::this_coro::executor;

  // 同时启动 2 个 start_acceptor.
  for (int n = 0; n < 2; n++) {
    asio::co_spawn(
      executor,
      start_acceptor(acceptor),
      asio::detached);
  }

  co_return;
}

asio::awaitable<void> start_acceptor(tcp::acceptor& acceptor)
{
  boost::system::error_code error;
  auto executor = co_await asio::this_coro::executor;

  while (true) {
    tcp::socket socket(executor);

    co_await acceptor.async_accept(
      socket, net::use_awaitable[error]);

    if (error) {
      // error 处理...
      break;
    } else {
      // 一些必要的业务流程.
      // ...
      // 创建一个 client_proc 协程处理 socket 上的数据传输.
      asio::co_spawn(
        executor,
        client_proc(std::move(socket)),
        asio::detached);
    }
  }

  co_return;
}
```
这样将同时发起 2 个协程来进行接受客户端连接请求，当然也可以更多个也没关系，具体根据情况调整。
