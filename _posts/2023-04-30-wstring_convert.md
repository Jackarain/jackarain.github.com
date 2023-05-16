---
layout: post
title: 浅谈 std::wstring_convert 及 utf 编码转换
---

遇到了一个字符串编码问题，因为只是简单的 `utf8/utf16` 之间的转换，所以我没有打算依赖三方库，于是 google 了一圈看看 c++ 本身有没有什么解决方案，我找到了标准库中提供的 `std::wstring_convert`，简单看了一下，API 的设计并不复杂，使用起来也很方便，但是一到 MS 平台，VC 一个劲的警告我， `std::wstring_convert` 已经被弃用，好吧，于是再找找看有没有替代器，结果是没有！！！啥玩意？挖坑不填？？`locale` 算你狠！

好吧，研究了一下 `std::wstring_convert` 到底是有啥问题，我看到微软的 stl github 上有一个 issues： https://github.com/microsoft/STL/issues/443 ，说是有内存泄露，还提供了测试程序：
```c++
#define _SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING
#include <assert.h>
#include <locale>
#include <stdio.h>

int fancyAlive = 0;

class fancy_state {};
class fancy_cvt : public std::codecvt<wchar_t, char, fancy_state> {
public:
  explicit fancy_cvt(size_t refs = 0)
      : std::codecvt<wchar_t, char, fancy_state>(refs) {
    ++fancyAlive;
  }
  ~fancy_cvt() {
    --fancyAlive;
    puts("destroyed!");
  }

  fancy_cvt(const fancy_cvt &) = delete;
  fancy_cvt &operator=(const fancy_cvt &) = delete;
};

int main() {
  {
    fancy_cvt *p = new fancy_cvt(5);
    assert(fancyAlive == 1);
    std::wstring_convert<fancy_cvt> cvt(p);
  }
  assert(fancyAlive == 0);
  puts("main() ending");
}
```
通过这段代码可以看到，`std::codecvt` 模板类内部有一个引用计数，`std::wstring_convert` 如果通过外部传递 `std::codecvt` 对象指针，那么它将在内部构造 `std::locale` 对象时直接使用外部传递的 `codecvt` 指针，当 `std::wstring_convert` 内部的 `std::locale` 析构的时候，`std::locale` 将自减它所保持的 `std::codecvt` 指针对象的引用计数，如果此时 `std::codecvt` 的引用计数不为 0，说明 `std::codecvt` 对象被 `std::locale` 之外持有，当然也不会析构 `std::codecvt`。

看到这里大致明白了 `std::wstring_convert` 的问题所在，这个问题的解决方案，实际上只需要 `std::wstring_convert` 禁止外部传递 `std::codecvt` 指针，又或者外部直接传 `std::locale` 对象指针给 `std::wstring_convert` 对象，就可以解决，但无论如何做都会导致 ABI 或 API 变动导致兼容性问题，总之最后 `std::wstring_convert` 结果就是被弃用了，并无替代方案，但并不是 `std::wstring_convert` 目前不能用，只是使用它会收到一堆警告，其实真正做事的东西是 `std::codecvt`，自己实现一下也很简单，这样就可以避免编译器警告了，下面是我的一些实现：
```c++
inline std::optional<std::u16string> utf8_utf16(std::u8string_view utf8)
{
	const char8_t* first = &utf8[0];
	const char8_t* last = first + utf8.size();

	std::u16string result(utf8.size(), char16_t{ 0 });
	char16_t* dest = &result[0];
	char16_t* next = nullptr;

	using codecvt_type = std::codecvt<char16_t, char8_t, std::mbstate_t>;

	codecvt_type* cvt = new codecvt_type;

	// manages reference to codecvt facet to free memory.
	std::locale loc;
	loc = std::locale(loc, cvt);

	codecvt_type::state_type state{};

	auto ret = cvt->in(
		state, first, last, first, dest, dest + result.size(), next);
	if (ret != codecvt_type::ok)
		return {};

	result.resize(static_cast<size_t>(next - dest));
	return result;
}

inline std::optional<std::u8string> utf16_utf8(std::u16string_view utf16)
{
	const char16_t* first = &utf16[0];
	const char16_t* last = first + utf16.size();

	std::u8string result((utf16.size() + 1) * 6, char{ 0 });
	char8_t* dest = &result[0];
	char8_t* next = nullptr;

	using codecvt_type = std::codecvt<char16_t, char8_t, std::mbstate_t>;

	codecvt_type* cvt = new codecvt_type;
	// manages reference to codecvt facet to free memory.
	std::locale loc;
	loc = std::locale(loc, cvt);

	codecvt_type::state_type state{};

	auto ret = cvt->out(
		state, first, last, first, dest, dest + result.size(), next);
	if (ret != codecvt_type::ok)
		return {};

	result.resize(static_cast<size_t>(next - dest));
	return result;
}
```
补充一点，不得不说，`utf16` 是个垃圾，因为取值范围导致它不能完整包含 `unicode` 编码，现在的 `unicode` 编码大致有110多万吧，而 `utf16` 通常在编程中使用固定的 16 位（`wchar_t`）的编码，所以装下几个字符编码？65536？所以它不能完整表示所有 `unicode` 字符编码，以及不能与 `utf8` 完全映射（`utf8` 编码空间巨大），而且 `utf16` 因为是双字节，存储/读取的时候，还需要注意字节序。

而 `utf8` 编码空间十分巨大，远超 `unicode` 而且没有字节序的问题，而且完美兼容 `ascii` 编码，但也由于它是变长编码，所以缺点就是在做字节串查找的时候，需要解析 `utf8` 编码后才能很好的做匹配。

另外：关于网上流传的 `std::wstring_convert` 确实存在问题，但主要是设计问题导致有可能内存泄露，而 `std::codecvt` 在处理 `utf8/utf16` 转换上是没有问题的，为了打消疑虑，我亲自验证了整个 `utf16` 编码所转换出的 `utf8`，没有发现任何问题。

