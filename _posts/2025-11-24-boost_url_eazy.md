---
layout: post
title: 使用 boost::url 定制 URL 解析器
---

在开发 `Proxy` 代理服务的过程中，用到了 `Boost.URL` 解析，因一次非标准的 `URL` 无法解析而出错，于是稍研究了一下 `Boost.URL` 设计，便决定通过 `Boost.URL` 定制一个简化的解析，`Boost.URL` 的设计非常优秀，只需要简单的实现一个符合规则的 `Rule` 就可以定制自己的解析器了，下面是我使用的代码示例：

```c++
	constexpr auto alpha = boost::urls::grammar::alpha_chars;
	constexpr auto digit = boost::urls::grammar::digit_chars;
	constexpr auto scheme_chars = boost::urls::grammar::lut_chars("+-.") + alpha + digit;

	// scheme, user, passwd, host, port, resource
	using url_info = std::tuple<
		std::string_view, // scheme
		std::string_view, // user
		std::string_view, // passwd
		std::string_view, // host
		uint16_t,         // port
		std::string_view  // resource (including the leading '/')
	>;

	struct urlinfo_rule_t
	{
		using value_type = url_info;

		boost::system::result<value_type> parse(char const*& it, char const* const end) const noexcept
		{
			std::string_view scheme, user, passwd, host, port, resource;
			char const* start = it;

			// 匹配 scheme: 字母开头，后跟字母、数字、+、-、.
			auto scheme_res = boost::urls::grammar::parse(it, end,
				boost::urls::grammar::token_rule(scheme_chars));
			if (!scheme_res)
				return boost::urls::grammar::error::invalid;

			scheme = *scheme_res;
			uint16_t port_num = boost::urls::default_port(boost::urls::string_to_scheme(scheme));

			// 匹配 "://"
			auto r = boost::urls::grammar::parse(it, end,
				boost::urls::grammar::literal_rule("://"));
			if (!r)
				return boost::urls::grammar::error::invalid;

			// 尝试解析 authority 部分，包括可选的用户信息和host和port信息.
			// authority = [ userinfo "@" ] host [ ":" port ]
			// authority 部分结束于第一个 '/' 或字符串结尾, 因此我们需要首先找到这个位置.
			// 计算 authority 部分的结束位置.
			char const* authority_start = it;
			char const* authority_end = it;
			while (authority_end < end && *authority_end != '/')
				++authority_end;

			// 确保 authority 部分不为空.
			if (authority_start == authority_end)
				return boost::urls::grammar::error::invalid;

			std::string_view authority(authority_start, authority_end - authority_start);

			// 解析 authority 部分, 如果有用户信息(包含’@‘则表示有用户信息), 则提取用户和密码.
			char const* host_start = authority_start;
			auto found = authority.find_first_of("@");
			if (found != std::string_view::npos)
			{
				host_start = authority_start + found + 1;

				char const* user_start = authority_start;
				char const* user_end = authority_start + found;
				user = std::string_view(user_start, user_end - user_start);

				found = user.find_first_of(":");
				if (found != std::string_view::npos)
				{
					char const* passwd_start = user_start + found + 1;
					char const* passwd_end = user_end;

					passwd = std::string_view(passwd_start, passwd_end - passwd_start);
					user_end = user_start + found;

					user = std::string_view(user_start, user_end - user_start);
				}
			}

			// 解析 host 和 port
			char const* host_end = authority_end;
			host = std::string_view(host_start, host_end - host_start);

			found = host.find_first_of(":");
			if (found != std::string_view::npos)
			{
				char const* port_start = host_start + found + 1;
				char const* port_end = host_end;

				port = std::string_view(port_start, port_end - port_start);
				if (!port.empty())
				{
					try {
						port_num = static_cast<uint16_t>(std::stoul(std::string(port)));
					}
					catch (...) {
						return boost::urls::grammar::error::invalid;
					}
				}

				host_end = host_start + found;
				host = std::string_view(host_start, host_end - host_start);
			}

			// 解析 resource 部分, 包括路径和查询参数和所有参数.
			resource = std::string_view(authority_end, end - authority_end);

			// 移动迭代器到末尾.
			it = end;

			return url_info{ scheme, user, passwd, host, port_num, resource };
		}
	};

	// 规则实例
	constexpr urlinfo_rule_t urlinfo_rule{};

	// parse_urlinfo 返回解析的 url_info 信息
	// 如果失败，则错误信息包含在 boost::system::result 的 error 中
	boost::system::result<url_info> parse_urlinfo(std::string_view s) noexcept
	{
		return boost::urls::grammar::parse(s, urlinfo_rule);
	};
```

在上面代码中，定义了 `urlinfo_rule_t` 这个类，并使用 `using` 设定其 `value_type = url_info`，这样解析后的结果将返回一个 `boost::system::result<url_info>` 的类型，而在上面我的 `url_info` 定义仅仅是简单的几个 `std::string_view` 的结构体。

然后就是实现 `urlinfo_rule_t` 的 `parse` 解析部分，这部分会相对复杂一点，除了自己通过迭代器指针解析之外，`Boost.URL` 也提供了一些 `grammar` 抽象概念，通过这些概念，甚至可以嵌套其它解析器组合起来实现想要的解析器，最后通过定制的函数 `parse_urlinfo` 来调用定制的 `Rule` 的解析器工作。

`Boost.URL` 严格按 `RFC3986` 规范编写，所以如果一些浏览器中能使用的 `URL` 通过 `Boost.URL` 未必能解析，这是因为浏览器是根据更宽松的名叫 `WHATWG URL Standard` 的 `URL` 规范，不得不说 `web` 相关的领域很多规范实在是太乱糟糟了。

最后必须称赞 `Boost.URL` 是相当不错的设计。
