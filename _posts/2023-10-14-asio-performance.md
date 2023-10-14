---
layout: post
title: 关于 c++ asio 性能小记
---

`c++ asio` 是一个具有极高性能的网络编程库，因为它完全遵循了 `c++` 的 0 开销设计原则，但如果只是参照 `asio` 的 `examples` 来实现一些服务，往往有可能在一些情况下并不能发挥出系统的所有性能，下面是我使用 `asio` 编写高性能服务一些实践中的经验。

一、尽量使用 `io_context` 池，并将每个 `io_context` 运行在单线程中，这样可以完全以单线程编程来处理 `io` 事件，从而避免锁，但需要注意 `io_context` 池均匀分配给 `io` 对象（通常使用 `round robin` 算法），避免线程发生饥饿的情况。在 `io_context` 构造时，需要注意设置恰当的 `concurrency_hint` 参数（ 参考文档: [https://www.boost.org/doc/libs/1_83_0/doc/html/boost_asio/overview/core/concurrency_hint.html](https://www.boost.org/doc/libs/1_83_0/doc/html/boost_asio/overview/core/concurrency_hint.html)），这有非常助于提高性能。

二、定制 `allocator` 以避免 `asio` 内部 `handler` 的 `op` 对象动态内存分配。

三、非紧急事件投递可使用 `defer` 而非 `post`。

四、在编写 `tcp server` 时 `listen` 设置恰当的 `backlog`（默认使用 `asio::socket_base::max_listen_connections` 即可）。另外重要的是，同时多次投递 `async_accept` 十分有助于大量客户端频繁连接，单个 `async_accept` 往往会在响应 `handle_accept` 之时，此时无法及时响应客户端的 `connect` 请求。
参考文章：[https://www.jackarain.org/2023/06/14/asio-acceptor-performance.html](https://www.jackarain.org/2023/06/14/asio-acceptor-performance.html)

五、在使用 `asio` 进行 `udp` 异步编程时，同时多次调用 `async_receive_from` 可大大提高吞吐量。

六、使用 `asio::io_context::executor_type` 而非 `asio::any_io_executor` 从而避免多态带来的开销。

七、使用 `io_context.poll()` 而非 `io_context.run()`，可以牺牲 `CPU` 来提高响应速度。

八、慎用 `strand`，如无必要不要盲目使用 `strand`，它主要是被设计在多线程环境中（通常是单个 `io_context` 运行在多线程），可以保证在其上的异步操作串行化而非加锁，而它在单线程中则没有必要。

九、传输大量数据时，使用自定义的 `transfer_at_least` 来提升效率，参考文章：[https://www.jackarain.org/2023/06/13/asio-transfer_at_least-performance.html](https://www.jackarain.org/2023/06/13/asio-transfer_at_least-performance.html) 


暂时只想到这些，以后想到了再更新，以上可根据需要做取舍。

