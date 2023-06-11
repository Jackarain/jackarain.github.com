---
layout: post
title: 基于 asio 异步机制实现目录监控
---
代码实现在 `dr` 实现的基础上修改而来，完善了很多处细节以及代码规范，详细代码见：

https://github.com/Jackarain/autogit/tree/master/autogit/include/watchman

整个 `watchman` 跨 `linux/windows` 平台，`windows` 平台使用 `ReadDirectoryChangesW API` 完成目录监控，`linux` 平台使用 `inotify API` 实现目录监控，该实现而且只需要简单的复制头文件即可使用，具体使用方法:

首先包含头文件：
```c++
#include "watchman/watchman.hpp"
```
然后在需要的地方创建 watcher 对象：
```c++
watchman::watcher dirmon(executor, boost::filesystem::path(dir));
```
之后就可以简单的使用：
```c++
auto result = co_await dirmon.async_wait( net::use_awaitable );
```
上面为 `c++20` 协程使用方法，也可以用其它形式来使用，比如使用 `callback` 或 `stackful` 协程，其返回的 `result` 即包含了目录变动详细。
