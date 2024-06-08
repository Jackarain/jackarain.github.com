---
layout: post
title: asio c++20协程的 redirect_error 用法改善
---

使用 asio 的 redirect_error 可以方便的在 c++20 协程中使用 error_code 而不是异常，如：

```c++
boost::system::error_code ec;
co_await timer.async_await(redirect_error(use_awaitable, ec));
```

这很好，唯一的缺点就是写起来有点冗长，不及 asio 中 stackful 协程的方括号写法更优雅，如：

```c++
boost::system::error_code ec;
timer.async_await(yield[ec]);
```

为了改善 asio 中 c++20 协程使用 redirect_error 更方便，我对 redirect_error 做了一个小小的包装，这样用起来就和 stackful 协程用法基本相同了，包装代码如下：

```c++
//
// Copyright (c) 2019 Jack (jack dot wgm at gmail dot com)
//
// Distributed under the Boost Software License, Version 1.0. (See accompanying
// file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
//

#pragma once

#include <boost/type_traits.hpp>
#include <boost/asio/io_context.hpp>
#include <boost/asio/awaitable.hpp>
#include <boost/asio/use_awaitable.hpp>
#include <boost/asio/redirect_error.hpp>

namespace asio_util
{
 template <typename Executor = boost::asio::any_io_executor>
 struct asio_use_awaitable_t : public boost::asio::use_awaitable_t<Executor>
 {
  constexpr asio_use_awaitable_t()
   : boost::asio::use_awaitable_t<Executor>()
   {}

  constexpr asio_use_awaitable_t(const char* file_name,
   int line, const char* function_name)
   : boost::asio::use_awaitable_t<Executor>(file_name, line, function_name)
   {}

  inline boost::asio::redirect_error_t<
   typename boost::decay<decltype(boost::asio::use_awaitable_t<Executor>())>::type>
   operator[](boost::system::error_code& ec) const noexcept
   {
    return boost::asio::redirect_error(boost::asio::use_awaitable_t<Executor>(), ec);
   }
  };
}

//
// net_awaitable usage:
//
// boost::system::error_code ec;
// stream.async_read(buffer, net_awaitable[ec]);
// or
// stream.async_read(buffer, net_awaitable);
//

// Executor is any_io_executor
[[maybe_unused]] inline constexpr
 asio_util::asio_use_awaitable_t<> net_awaitable;

// Executor is boost::asio::io_context::executor_type
[[maybe_unused]] inline constexpr
 asio_util::asio_use_awaitable_t<
  boost::asio::io_context::executor_type> ioc_awaitable;

```

做成一个头文件，并在使用前 include 它，则可以在协程中这样使用：

```c++
boost::system::error_code ec;
co_await timer.async_await(asio_util::use_awaitable[ec]);
```

这样，酷似 asio 中 stackful 协程的方括号用法了，可以在处理 error_code 时重复使用同一 ec 变量。
