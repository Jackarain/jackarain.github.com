---
layout: post
title: 愚蠢的 on('data', cb) 接口设计
---

愚蠢的 `on('data', cb)` 接口设计，在所有语言的 io 库中，但凡是类似这种设计的 API，都是十分愚蠢的设计。

正确的设计应该是类似 asio 中的 `async_op(cb)`，其中 cb 只会被响应单次，除非在 cb 中重新发起 `async_op(cb)`，当然也可发起其它异步操作。

因为 `async_op(cb)` 更为灵活，且包含了 `on('data', cb)` 这种用例。

另外由其当需要将这种异步方法转换成协程支持的时候，就能更加体现出它的优势所在了。

而不是在 `on('data', cb)` 中处理接收到的数据时，不停的判断接收到的数据是属于哪个流程，这既繁琐又易出错，这由其是在处理一些较复杂的应答式协议时更加糟糕无比，比如：
``` c++
void onread(data) { // onread 就是 on('data', onread)，接收到数据将被调用
  if (state == login) {
    on_process_login(data); // 当前为login状态，则按login逻辑处理接收到的data
    return;
  } else if (state == getwork) {
    on_process_getwork(data); // 当前为getwork状态，则按getwork逻辑处理接收到的data
    return; 
  } else if ... // 其它状态的判断...
}
```
上面这种模式十分糟糕，除了需要用户维护复杂的 state 状态之外，其请求发送部分代码被迫割裂在其它地方运行，而接收响应部分被统一到了一个 onread 函数当中，这是十分糟糕的。

正确的打开方式，通常像这种很多这种应答式协议流程应该用协程来处理，将会十分优雅：
``` c++
void start_work() {
  await login(request);
  await read(data);
  // 处理server返回的data...然后继续后面的工作
  await getwork(request);
  await read(data);
  // 处理data，根据work完成业务
  ...
  // 继续使用协程以最自然的同步流程方式处理后续流程，而不需要 state 保存状态...
  ...
}
```
即便是那种简单的一问一答被动响应式的协议处理，在只处理请求的 server 端中的事件响应，也可以以下方式通过运行一个循环来异步读取就行了，而不必是 `on('data', onread)` ：
``` c++
void work_loop() {
  while (true) {
    data = await read();
    if (data.type == request1) on_request1(data);
    if (data.type == request2) on_request2(data);
    ...
  }
}
```
综上可看出，`on('data', onread)` 这类接口十分难以使用在这种复杂的应答协程，也很难转换成协程接口（转换成协程这个工作不适合在用户代码中去做库本该做的工作），其次状态机的维持极其容易出错。

`async_op` 这种类型的接口则很容易转换成协程，因为 IO发起 与 IO完成，能轻松一一对应到协程的 `suspend` 和 `resume`，即便不是协程也可亦或是 `Resumable` 函数，这也会得到一个较为清晰的业务流程。

简而言之，异步 io 库的 api 接口设计，不应该设计为 `setInterval`，而应是 `setTimeout`，但凡只设计为 `setInterval` 这种方式的 io 接口，都是十分糟糕的设计。

当然，与其说是 `setInterval` 和 `setTimeout` 的区别，更应该说是事件驱动模型和异步操作模型的区别，一个库在实现IO接口时，应当尽可能设计能作为异步操作模型为基础的接口，或者直接设计为异步接口。

事件驱动模型在处理流程复杂的应答式协议时，需要维护状态机，使代码难以理解和维护；而异步操作模型则更加灵活，可以通过协程等方式将异步操作转换成同步代码，使代码结构更加清晰易懂。

