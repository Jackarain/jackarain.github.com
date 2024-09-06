---
layout: post
title: move *this 在自定义异步子程序中的思考
---

昨天给 `boost.beast` 提交了一个 `bug`，出问题的代码如下：

``` c++
     net::dispatch(this->get_immediate_executor(),
         net::append(std::move(*this), ec, 0));
```

这是在一个 [transfer_op](https://github.com/boostorg/beast/blob/21545dbcaf22d21ba72c0021dae0ee85c7004fc5/include/boost/beast/core/impl/basic_stream.hpp#L328-L330) 的代码中，用于发起 `net::dispatch` 再回调到自己的 `operator()(...)` 中，其中 `std::move(*this)` 实际上就是重新通过 `move` 构造一个新的 `transfer_op` 来充当回调的函数对象，问题就是因为在 `std::move` 操作之后，`this->get_immediate_executor()` 将会变得无效，这是因为函数调用时的求值顺序通常是从右往左的，这就会发生未定义行为，我在提交了这个 [Issues](https://github.com/boostorg/beast/issues/2925) 之后，`boost.beast` 维护者几乎同时就确认了问题并修复了该错误。

像这种错误代码很容易不小心就写出来，在这事后，我意识到像这种问题也许不只有这一种情况，因为在 `boost.asio` 或 `boost.beast` 中，经常会遇到很多特定的一些异步操作组合，如 `async_read_until`, 它实际上内部是由一个或多个异步 `read` 操作的组合，直到满足条件后才通过回调返回，像这样的操作，通常内部会实现为一个下面这样的形式：

```c++
template<class Stream, class DynamicBuffer, class Condition, class Handler>
class read_op
{
    Stream& stream_;
    DynamicBuffer buffer_;
    Handler handler_;
    std::size_t bytes_transferred_ = 0;
    int start_ = 0;

public:
    […]

    void operator()(error_code ec, std::size_t bytes_transferred = 0, int start = 0)
    {
        switch (start_ = start)
        {
        case 1:
            for(;;)
            {
                stream_.async_read_some(std::move(buffer_), std::move(*this));
                return; default:

               if (ec || bytes_transferred == 0)
                   break;

                bytes_transferred_ += bytes_transferred;
                if(Condition{}(buffer_))
                   break;
            }
            handler_(ec, bytes_transferred_);
        }
    }
};
```

其中：

```c++
stream_.async_read_some(std::move(buffer_), std::move(*this));
```

是不是也有同样的问题呢？也就是说 `buffer_` 这个发生在 `this` 所在的对象被 `move` 之后，`buffer_` 还是否有效？

以后有空再续这个话题吧。。。
