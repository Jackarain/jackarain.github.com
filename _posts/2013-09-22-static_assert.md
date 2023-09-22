---
layout: post
title: c++ 中 if constexpr 的 always_false
---

在模板编程中，经常会用到 `if constexpr` 用来做类型判断，实现不同类型的代码匹配，所以当模板传的不支持的类型的时候，通常会使用 `static_assert` 来实现编译时断言，通常情况如下：
```c++
template <typename T>
void foo() {
    if constexpr (std::is_same_v<T, int>) {
        //...
    } else if constexpr (std::is_same_v<T, float>) {
        // ...
    } else {
        static_assert(...);
    }
}
```
那么上面的 `static_assert` 应该怎么写呢？`static_assert` 的判定条件需要依赖类型 T，所以需要实现一个在 `else` 中始终为 `false` 的 `constexpr` 条件表达式，所以这里直接写 `false` 肯定不行，通常定义为：
```c++
template <typename T> struct always_false : std::false_type {};
```
或
```c++
template<class ... T> inline constexpr bool always_false = false;
```
这样使用的时候，就可以像下面这样使用：
```c++
static_assert(always_false<T>::value, "...");
```
或
```c++
static_assert(always_false<T>, "...");
```
这个语义十分清楚，但这也有一个缺点，因为这个 `always_false` 一般会被定义在项目中的一个公共头文件中，整个项目中都会使用到它，这时如果引入的一个三方项目，很容易碰到相同的 `always_false` 定义，这个时候就会有点麻烦了，那么怎么做会避免这种情况发生呢？很简单，不再自己定义 `always_false`，我的解决方法如下：
```c++
static_assert(!std::is_same_v<T, T>, "...");
```
其中条件为 `!std::is_same_v<T, T>` 这样它同 `always_false` 一样始终将为 `false` 的常量表达式，且不再需要自己去定义那个到处被用到的 `always_false` 类型，唯一不足的是降低了一点可读性。
