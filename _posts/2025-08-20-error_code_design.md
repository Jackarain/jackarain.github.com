---
layout: post
title: 论 c++ 中 error_code 的设计
---

这篇文章是我在知乎上回复关于 `error_code` 的提问整合而来的，`error_code` 这套东西是出自 `Beman Dawes`(`boost` 库发起者, 已经去世了) 与 `Christoper Kohlhoff`(`asio` 作者) 等一众大佬的设计。

`error_code` 的设计雏形之初是 `Kohlhoff` 为了解决 `asio` 中系统错误代码和自定义错误代码的统一所做的抽象设计。

试想，编写一个网络库，要如何将系统错误码，在不丢失信息的情况下（也就是保留系统错误代码，比如 `windows` 上 10061 代码表示 “由于目标计算机积极拒绝，无法连接。 ” 这2个信息都不能丢失）如实的汇报给调用者。

还有库特定的错误代码，也要能在不增加使用者负担的情况下，以同样的方式给调用者使用。

另外在不同操作系统上不同错误代码，还要可以根据语义上的统一，去使用 `error_code` / `error_condition`，比如 `operation_aborted` 它无论在 `windows` 还是 `linux` 或 `macos` 中或其它系统，语义上都是相同的，无论实际的系统错误代码值是什么值，像这种抽象设计就可以在编程中根据语义来判定错误代码，从而去做相关的业务逻辑处理，而不需要使用者去区分什么操作系统啊、错误码是啥，等等这些麻烦事。

`error_code` 在进入 `boost` 之后，就被广泛使用，再后来都不需要 `Kohlhoff` 出手(实际是由 `Beman` 大佬等人推动的)，就顺理成章的进入了 `c++11` 标准。

在我第一次看过它的设计之后我就喜欢上了, 并且一直在使用, 它有很多优点, 包括上面提到的，还有比如不必担心错误代码值被重复定义(可以通过自己定义的不同 `error_category`), 有更可读的错误信息(`message`)而不止是一个数值码。

在 `error_code` 的设计中，其中可能让人难以理解的就属 `error_condition`，具体而言，`error_condition` 就是为了实现可移植性的 `error_code`，如 `no_such_file_or_directory` 就是不同平台上的相同错误类型，但错误代码 `value` 并不一定相同，例如 `std::filesystem::copy` 函数的错误处理：

```c++
std::error_code ec;
std::filesystem::copy(from, to, ec);

if (ec == std::errc::no_such_file_or_directory) {
  // ...
}
```

在上面代码中

```c++
if (ec == std::errc::no_such_file_or_directory)
```

`std::errc::no_such_file_or_directory` 则是与平台无关的 `error_condition`，而 `ec.value()` 是平台相关的错误代码，也就是操作系统返回的错误值。

而如果写成：

```c++
if (ec.value() == ERROR_PATH_NOT_FOUND)
```

则不具可移植性，仅在 `windows` 下有效（`ERROR_PATH_NOT_FOUND` 是 `windows` 上的具体错误代码定义）。

简而言之，`error_code` 是具体的错误值，通常包含的信息就是操作系统返回的，而 `error_condition` 是抽象的与平台无关的错误表示。

再举一个比较易懂示例就是，比如在一些平台中可能 `EAGAIN` 并不等于 `EWOULDBLOCK`，但是这2个错误值完全表达了一个意思，所以，无论 OS 返回的 `ec.value()` 是 `EAGAIN` 或是 `EWOULDBLOCK`，都对应到 `std::errc::resource_unavailable_try_again` 这个 `error_condition`，你不需要关心 `EAGAIN` 和 `EWOULDBLOCK` 在某些系统上是否是同一个值，只需要判断

```c++
if (ec == std::errc::resource_unavailable_try_again)
```

这样就包含了 `EAGAIN` 和 `EWOULDBLOCK` 两种情况，完整示例代码：

```c++
#include <iostream>
#include <system_error>
#include <unistd.h> // 包含 EAGAIN 相关定义

int main()
{
    {
        // 这里模拟产生 EWOULDBLOCK 的系统错误
        std::error_code ec(EWOULDBLOCK, std::system_category());

        if (ec == std::errc::resource_unavailable_try_again) {
            std::cout << "resource_unavailable_try_again\n";
        } else {
            std::cerr << "其他错误: " << ec.message() << std::endl;
        }
    }

    {
        // 这里模拟产生 EAGAIN 的系统错误
        std::error_code ec(EAGAIN, std::system_category());

        if (ec == std::errc::resource_unavailable_try_again) {
            std::cout << "resource_unavailable_try_again\n";
        } else {
            std::cerr << "其他错误: " << ec.message() << std::endl;
        }
    }

    return 0;
}

```

当然，包含在 `std` 中的 `error_code` 并不完整，若 `std` 中没有定义的话，就得自己扩展，关于 `error_code` 扩展以后有空再写吧。。。