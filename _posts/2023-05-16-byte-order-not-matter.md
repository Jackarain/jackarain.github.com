---
layout: post
title: 协议实现不应该关心字节序
---

这个标题虽说看起来有点反常理和直觉，尤其是对于一些网络老手来说，但这个标题并不是我首先提出来，在这之前已经有不少大佬在很多年前就提出来了，只是网上很少有相关的讨论。

之所以我想到这个话题，是因为最近我的一个网络服务器更新了之后，同事发现客户端不能在`MIPS`正常工作，因为他所跑的`MIPS`是大端序，所以他怀疑可能是字节序问题，在这个问题上，我持不同意见，因为协议以及协议编码及解析，采用了字节序无关的设计，在这里，很多人可能不太清楚什么是字节序无关，这就是我要说的主题，当然最终那个问题得已修复，确实是其他问题导致的。

协议通常包括字符串，数值，一般像字符串是不存在字节序的，编码是如何存的字符串，解码时依次读取就好，但是数值在人类眼里，往往不同于存储在机器里面，如 `0x0307` 这样一个2字节的短整型数值(`uint16_t`)，其在小端序的机器上，在内存中以：

`0x07 0x03`

存储，若是在大端机器上，则是以：

`0x03 0x07`

存储，无论是大端序还是小端序，它们都是表示 `0x0307` 这样一个数值，我们可以对其做任何运算，它也是以 `0x0307` 来计算的，那么，现在我们可以这样对它进行位移运算：

``` bash
uint16_t x = 0x0307;
uint8_t one_byte = (uint8_t)(x >> 0) & 0xff; // one_byte == 0x07
uint8_t two_byte = (uint8_t)(x >> 8) & 0xff; // two_byte == 0x03
```

那么这 2 个计算，无论是在大端机器还是小端机器上，其计算结果 `one_byte` 及 `two_byte` 是确定的，永远都是上面这个结果。若将 `0x0307` 扩展到 `uint32_t` 或 `uint64_t`，其结果也是一样的，比如我在写 `socks` 时通过模板设计了一个较为通用的实现：( https://github.com/Jackarain/proxy/blob/master/proxy/include/proxy/socks_io.hpp )

``` c++
template<typename type, typename source>
type read(source& p)
{
 type ret = 0;
 for (std::size_t i = 0; i < sizeof(type); i++)
  ret = (ret << 8) | (static_cast<unsigned char>(*p++));
 return ret;
}

template<typename type, typename target>
void write(type v, target& p)
{
 for (auto i = (int)sizeof(type) - 1; i >= 0; i--, p++)
  *p = static_cast<unsigned char>((v >> (i * 8)) & 0xff);
}
```

我见过很多程序当中，根据本机大小端使用 `BIG_ENDIAN、LITTLE_ENDIAN` 这种宏，然后分别做类似：

``` c++
uint16_t x = 0x0307;
#ifdef LITTLE_ENDIAN
uint8_t one_byte = *(uint8_t*)&x; // 取x变量地址上第1个字节，0x07
uint8_t two_byte = *(((uint8_t*)&x) + 1); // 取x变量地址上第2个字节，0x03
#else
uint8_t two_byte = *(uint8_t*)&x; // 取x变量地址上第1个字节，0x03
uint8_t one_byte = *(((uint8_t*)&x) + 1); // 取x变量地址上第2个字节，0x07
#endif
```

上面这种方法，是完全根据机器的字节序，按存储在内存上的顺序，分别在不同地址上获取相应 `byte`，这样才能正确的拿到确定的数据，很明显，这是极为容易出错的，并且代码十分的丑陋和冗长，即使把这些操作定义成翻转的宏或函数，也徒增了对机器架构的依赖。

综上所述，我们在设计可移植协议时，编程实现的时候不应该关心机器的字节序，而应该按数值的方式来计算正确的数据，也就是按我上面定义的 read/write 函数来进行数值的序列化，不用关心是 `big endian` 或 `little endian` 还是 `mixed endian`。

很多人肯定会非常抗拒这种写法，其理由是：“这里有个 for 循环，还要位移好几次，性能一定不如直接指针强转或者 bswap/ntohl 之类的函数吧？” 这是对现代优化编译器的最大误解，实际上它是 0 运行时开销！

在开启 -O2 或 -O3 优化后，编译器拥有极强的模式匹配（Pattern Matching）能力，当它看到你对一个 sizeof(type) 大小的循环进行 `<< 8` 和 `|` 操作，它能立刻洞悉你的真实意图，它会在小端序机器上，编译器会自动把整个 read 循环优化成一条 MOV 指令加上一条 BSWAP（字节翻转）指令，或者直接一条带翻转的加载指令。在大端序机器上，编译器发现网络序和本机序一致，整个循环会被直接优化掉，变成一条简单的 MOV 指令。

有兴趣的朋友通过 godbolt 验证：[godbolt](https://godbolt.org/z/j6T1513oW)

```c++
parse_uint16(unsigned char const*):                    # @parse_uint16(unsigned char const*)
        movzx   eax, word ptr [rdi]
        rol     ax, 8
        ret

parse_uint32(unsigned char const*):                    # @parse_uint32(unsigned char const*)
        mov     eax, dword ptr [rdi]
        bswap   eax
        ret
```

当然现在来说，一般都是不再亲自设计底层协议，而是直接利用一些开源且使用广泛的库来做这些事，比如二进制协议一般采用 `boost.endian，protobuf，flatbuffers`，或者采用文本协议如 `json`，甚至是古老的 `xml`，这些都是不用程序员关心字节序的问题的，但是这些在要求比较高的场合，并不能满足要求，比如 `protobuf` 性能不是太好，`flatbuffers` 占用存储空间有点大，文本协议更是如此，这种情况下，还是不得不根据自己的业务设计合适的协议，这时对这些知识有所了解是十分有意义的。
