---
layout: post
title: 关于协程是否需要协程锁的思考
---

这个问题我在很多年前就思考过并有了下面的结论, 但是一直没有总结成文, 今天突然看到一个贴子上讨论这个问题, 于是回答的同时总结了一下我多年前关于这个话题的相关思考

协程和确实锁完全没有关系, 因为锁通常意义指用于处理并发冲突(无论是多线程的锁, 还是数据库的锁, 都是处理并发访问带来的冲突问题), 协程本身并不存在并发, 所以并没有这个意义的上锁.

当然一些人啊, 把异步执行流程上的冲突问题, 也归结到锁的问题, 比如下面代码:

```c++
std::vector<int> data;

net::awaitable<void> proc_data() {
  // 此处如果不加锁, proc_data 被同时调用2次以上, 将有可能导致下面 data 的迭代器失效.
  // co_await lock(net_awaitable[ec]); // 如果此处有这样一个锁, 那么就可以避免这个问题

  for (auto it = data.begin(); it != end();) {
    co_await async_write_data(*it);
    it = data.erase(it); // 从容器中删除
  }
}
```

又或者以下代码:

```c++
net::awaitable<void> proc_data(tcp_socket& sock) {
  error_code ec;

  char bufs[1024];
  // 由于 tcp_socket 不能同时发起 2 次 async_read 操作(这也属于UB)
  // 所以函数 proc_data 被同时调用2次就是UB.

  // co_await lock(net_awaitable[ec]); // 如果此处有这样一个锁, 那么就可以避免这个问题.

  co_await net::async_read(sock, net::buffer(bufs), net_awaitable[ec]);
  co_return;
}
```

看起来协程虽然是单线程, 也很需要 '锁' 的嘛, 不是吗? 我个人认为并不是, 这本质上属于不可重入, 非要写成了重入, 对于不能瞎调用函数瞎调用了属于是, 这导致的问题, 其实是自己代码写的有问题导致的, 程序员应该正确的设计程序流程, 应该了解所调用的函数的规则.

我的理由和不推荐使用递归锁一样, 使用递归锁看似很爽对吧, 通常是属于设计出了问题,才会出现需要递归锁, 需要注意的是 c++ 虽然也提供了std::recursive_mutex, 但没人推荐使用它, 而且它并不是安全的, 在同一个线程可以连续获得锁的次数是不确定的, 超出将会抛出异常.

提供种级别东西的 '锁', 只会使得喜欢乱来的人更爱乱来, 你是想堆屎山吗? 如果是想堆屎山, 那就当我没说...

为了更深入思考这个问题的本质, 接下来, 让我们继续以第1个为示例, 我将它稍做修改, 这次将上面使用协程的代码展开成不使用协程的代码:

```c++
void proc_data() {
  auto it = data.begin();
  proc_data_impl(it);
}

void proc_data_impl(auto it) {

  if (it == data.end())
    return;

  // 针对 data 元素的异步操作函数
  async_write_data(*it, [it]() mutable {
    it = data.erase(it); // 在 async_write_data 完成回调函数中, 从容器中删除
    proc_data_impl(it);
  });
}
```

请注意看, 上面 proc_data 已经不是一个协程函数了, 但它仍然是不可重入的函数, 如果在其它地方重复调用 proc_data, 一样会导致迭代器失效而出错, 请问各位, 这跟协程锁有关系吗? 还需要啥啥锁吗?

显然是跟协程锁没有半毛钱关系了, 且显然就是设计出了问题, 在一个不能重复调用内部有异步操作的函数上, 同时发起多次调用执行, 那就会出错.

有人跟我说 `golang` 中的 `goroutine` 就支持加锁此类操作, 但是我想说的是, `goroutine` 根本就不是传统意义上的协程, 它本质上算是和系统线程是一回事.
