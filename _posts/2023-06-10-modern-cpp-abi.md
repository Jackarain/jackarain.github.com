---
layout: post
title: 现代 c++ 解决 c++ abi 问题的一些方法
---

c++ abi 问题一直以来，都不能得到很好的解决，abi 导致的问题主要表现为内存布局发生改变，运行时可能随时进程崩溃。
最为常见的情况就是，一个c++写的lib导出的api接口，比如：
```c++
class libfoo {
public:
  void printx() const; // 打印输出成员变量x

private:
  int x{11};
};
```
而这个lib在发布新版本后，因为添加了新的成员变量，导致内存布局发生变化：
```c++
class libfoo {
public:
  void printx() const; // 打印输出成员变量x

private:
  int y{12};
  int x{11};
};
```
这将导致调用者在调用到printx时必然会发生UB而导致crash或怪异行为。

c++11 提供了一个叫作`inline namespace`的功能，利用这个功能，我们便可以做到，在保证lib的接口不变的情况下，调用者能在编译期检查出 abi 的不匹配，这样不至于在运行过程中 crash 掉程序。

比如我们在 lib 的第1个版本中实现为：
```c++
namespace lib {
 inline namespace v1 {
    class libfoo {
      public:
      void printx() const; // 打印输出成员变量x

      private:
      int x;
    };
 }
}
```
那么调用者只需要`lib::libfoo`就可以访问到`printx`，而v1这个`inline`的名字空间就好像不存在一样，当我们作为 lib 的作者更新了此库，比如更新为v2
```c++
namespace lib {
 inline namespace v2 {
    class libfoo {
      public:
      void printx() const; // 打印输出成员变量x

      private:
      int y;
      int x;
    };
 }
}
```
那么调用者如果不重新编译链接新的库，那么v2这个新版本的 lib 中因为 `lib::v1::libfoo` 相关的所有符号不存在，将会报告找不到，这样将避免了运行过程中UB导致出错，也从而使得调用者不得不使用新 lib 的声明重新编译调用程序，获得正确的内存布局和相关符号，也因为新版 lib 使用了`inline namespace`，也使得调用者不需要做任何修改，就可以无痛切到v2版本。

我的建议是每次库版本更新的同时，也更新`inline namespace`的名称，从而保证调用者始终跟 lib 同步而不出现UB。


