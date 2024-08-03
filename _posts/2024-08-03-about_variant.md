---
layout: post
title: C++ 之 variant 浅谈
---

在很久很久以前，还没了解到 `variant` 时，我在使用 `c++` 写一些网络应用时，会经常碰到一些这样需求，就是通过 `tcp` 去连接一个服务器时，这个服务器可能是 `ssl` 协议，也可能不是，就比如 `http/https` 客户端，这时只能通过在运行时根据用户提供的 `url` 的 `scheme` 来决定构造 `ssl_socket` 还是普通 `socket`，构造好之后，在使用方法上面就没有什么区别了，即通过 `socket.read` / `socket.write` 来读写数据。

最初我对此的抽象是这样的：

```c++
class http_client {
  /// 一些其它实现，略...

  void send_request() {
    if (is_https_) {
      ssl_socket_.write(request);
    } else {
      socket_.write(request);
    }
  }

  /// 这里定义了 is_https_ 来表示 http_client 正在处理的是 https 还是 http
  /// 在每次网络读写时，通过 is_https_ 来分别走不同的分支, 如上面的 send_request
  /// 中的 if 条件判断。
  bool is_https_;
  net::socket socket_;
  net::ssl_socket ssl_socket_;
};
```

很显然，这种实现方法十分笨拙，通过 `is_https_` 到处判断也导致代码十分臃肿难看，其中的 `socket_` 和 `ssl_socket_` 始终有一个是处于无用的状态，显得十分碍眼，我在当时自然想到了通过虚函数来实现这类抽象，解决方案大致如下：

```c++
class http_socket {
public:
  virtual int wirte(const request& req) = 0;
  virtual int read(response& resp) = 0;
};

class http_ssl_socket {
public:
  virtual int write(const request& req) override
  {
    return ssl_socket_.write(req);
  }

  virtual int read(response& resp) override
  {
    return ssl_socket_.read(resp);
  }

  net::ssl_socket ssl_socket_;
};

class http_nossl_socket {
public:
  virtual int write(const request& req) override
  {
    return socket_.write(req);
  }

  virtual int read(response& resp) override
  {
    return socket_.read(resp);
  }

  net::socket socket_;
};
```

这样在实现 `http_client` 时就可以简单的写成：

```c++
class http_client {
  /// 一些其它实现，略...

  void send_request() {
    socket_ptr_->write(request);
  }

  /// 只要在构造 http_client 时根据用户传入的 url 的 scheme 来创建
  /// http_ssl_socket 或 http_nossl_socket 即可，使用这个基类指针 socket_ptr_
  /// 指向所创建的对象即可。
  http_socket* socket_ptr_;
};
```

通过虚函数来抽象运行时多态的这种做法，在实现 `http_client` 时变得轻松不少，代码也更清晰简洁，虽然 `virtual` 是个让人感觉不太舒服的关键字，但这可能在当时已经是我的最优解了，直到我通过阅读 `libtorrent` 这个开源项目，了解到模板元编程，我找到了另一种方法：

```c++
template <
    BOOST_PP_ENUM_BINARY_PARAMS(2, class S, = boost::mpl::void_ BOOST_PP_INTERCEPT)
>
class http_socket {
public:
    typedef BOOST_PP_CAT(boost::mpl::vector, 2)<BOOST_PP_ENUM_PARAMS(2, S)> types0;
    typedef typename boost::mpl::remove<types0, boost::mpl::void_>::type types;

    typedef typename boost::make_variant_over<
        typename boost::mpl::push_back<
        typename boost::mpl::transform<
        types
        , boost::add_pointer<boost::mpl::_>
        >::type
        , boost::blank
        >::type
    >::type variant_type;

    /// 一些其它实现，略...
    variant_type m_variant = boost::blank();
};
```

说实话，上面这模板看着十分头痛也不优雅，然后通过实现各种 `visitor` 去访问 `m_variant`，大致如下：

```c++
template <class Request>
struct write_visitor
  : boost::static_visitor<std::size_t>
{
  write_visitor(Request const& req)
    : req_(req)
  {}

  template <class T>
  std::size_t operator()(T* p) const
  { return p->write(req_); }

  std::size_t operator()(boost::blank) const
  { return 0; }

  Request const& req_;
};
```

实现一堆这种 `visitor` 来访问 `m_variant`，虽然丑陋，但当时我认为算是找到了运行时多态另一种答案 `variant`，具体项目中使用参考，再后来自然而然就开始直接使用 `variant` 而不是这种复杂的模板元，如：

```c++
template<typename... T>
class base_stream : public boost::variant2::variant<T...>
{
    template <typename Request>
    auto write(const Request& req)
    {
        return boost::variant2::visit([&](auto& t) mutable
        {
            return t.write(buffers);
        }, *this);
    }

    /// 一些其它实现，略...
};
```

然后直接在 `http_client` 中使用 `base_stream<tcp_socket, ssl_stream>` 即可：

```c++
class http_client {
  /// 一些其它实现，略...

  void send_request() {
    socket_.write(request);
  }

  base_stream<tcp_socket, ssl_stream> socket_;
};
```

相比之前的版本要简洁多了，虽然依然需要实现 `visit`，但这已经不再是障碍，具体项目中使用可参考。

得益于 `variant` 的优秀设计和现代编译器的优化能力，也可以说是遵循了 `c++` 零开销原则，这里给出一个代码 [https://godbolt.org/z/KbsqE3nvv](https://godbolt.org/z/KbsqE3nvv) 用于参考和研究。

在这里，通常我更倾向于使用 `boost.variant2`，因为它不会处于无值状态（无值状态简而言之就是当具体类型构造时发生异常导致构造失败，这也导致 `variant` 处于这个类型的无值状态，这可以看成是错误的，占着茅坑的屎），而标准库的 `std::variant` 则是要求不能在初始化（构造或移动、复制构造）时抛出异常，否则它将进入 `valueless` 状态，可通过 [valueless_by_exception](https://en.cppreference.com/w/cpp/utility/variant/valueless_by_exception) 来检测这个状态。

当然，即使使用 `std::variant` 出现 `valueless` 也是小概率，因为绝大部分 `c++` 代码不会在各种构造中去抛异常，因为这有可能会导致资源泄露等很多其它问题，这也是 `c++` 程序员都很忌讳的事情。
