---
layout: post
title: Sender/Receiver 模式的设计缺陷
---

在 `Sender/Receiver` 模型中，我们知道它的设计基础是由几个看似简单的组合来实现的，也就是 `Sender`、`Receiver` 还有 `operation state` 以及 `connect` 这些概念构成，其中工作流程就是通过 `connect` 将 `Sender` 和 `Receiver` 组合起来，返回一个 `operation state`，运行这个 `operation state` 的 `start` 方法就可以驱动这个异步操作，并通过 `Receiver` 回调其中之一接口函数以获取执行的结果，这就是这个模型的主要工作过程。

然而在这个模型中，`Receiver` 被设计成具有如下3个回调函数的接口对象：

```bash
set_value 根据文档，set_value 被调用即表示执行成功后的结果
set_error 表示发生了错误
set_stopped 表示未发生错误也未执行成功，主要就是指工作被取消
```

有了以上基础的介绍，然后必须讲到在我开发我的 `proxy` 程序时，因为需要编写自己的数据接收，其中很多像 `async_read_until` 这种条件接收，如：

```c++
error_code ec;
bytes = co_await net::async_read_until(socket, buf, '\0', net_awaitable[ec]);
```

类似这种代码在使用 `asio` 进行网络编程时，其实非常普遍，它表示按 '\0' 为条件接收数据，直到接收的数据中遇到 '\0' 标识，则返回已接收到的数据，在这个过程中，有可能对方发送端在发送了一段数据后，就关闭了 `socket`，这意味着你并不总是能读取到 '\0' 这个异步读取就会返回，其中 `ec` 通常是 `eof` 或 `reset` 等，而 `buf` 则是在发生 `eof` 之前陆续接收到的所有数据，所有信息均在。

我说这个案例是想表达什么呢？这表示我们接收到了数据的同时也接收到了一个 `error` 标识，这在 `asio` 中是没有任何问题的，然而如果你若是使用 `Receiver` 模式来表达呢？你要么就是回调 `set_value`，要么就是回调 `set_error`，也就是说这种场景 `Receiver` 根本无法正常表达。

综上，我认为 `Sender/Receiver` 并不合适做为异步网络编程基础模型。`Sender/Receiver` 模型自从被提出来后被吹捧到一个非常高的高度，但却久久没有一个能基于它来实现的一个完善的异步网络库，这也许是其中原因之一？
