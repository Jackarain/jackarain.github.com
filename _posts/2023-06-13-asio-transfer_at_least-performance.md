---
layout: post
title: 关于在 asio 使用 transfer_at_least 数据传输时性能优化
---

在使用 `asio::async_read` 来接收大量数据的时候，使用 `transfer_at_least` 来设定最少接收数据量，但是 `asio` 默
认的 `transfer_at_least` 实现每次最大读取 `64k`, 这在传输大流量的时候，是极为不合适的，会极大的影响数据接收性能。
所以我们可以自己定制一个 `transfer_at_least` 来取代 `asio` 默认的 `transfer_at_least` 函数，实现如下：

```c++
  static std::size_t custom_default_transfer_size = 1024 * 1024;
	class transfer_at_least_t
	{
	public:
		typedef std::size_t result_type;

		explicit transfer_at_least_t(std::size_t minimum)
			: minimum_(minimum)
		{
		}

		template <typename Error>
		std::size_t operator()(const Error& err, std::size_t bytes_transferred)
		{
			return (!!err || bytes_transferred >= minimum_)
				? 0 : custom_default_transfer_size;
		}

	private:
		std::size_t minimum_;
	};

	inline transfer_at_least_t transfer_at_least(std::size_t minimum)
	{
		return transfer_at_least_t(minimum);
	}
```
上面代码中 `custom_default_transfer_size` 设置为 1M 大小用于默认接收缓冲大小，此时在 `asio::async_read` 函数中 `prepare` 将分配最大为 1M 的内存空间用于数据接收。


