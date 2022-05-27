---
title: "C++_tips"
date: 2022-05-27T23:29:05+08:00
draft: false
tags: ["C++","language"]
category: ["C++"]
author: "clundro"
---

本篇文章主要是对阅读 [google c++ tips](https://abseil.io/tips/) 后进行的总结与记录。

## tips 186: 函数请放在匿名空间中

默认命名空间即全局命名空间，`main()`必须在该空间中。其他命名空间调用默认空间函数时，需使用 `::`作用符号。

```cpp
void func1();
namespace a{
  void func2(){
    ::func1();
  }
}
void func1(){}
```

当新加一个函数时，默认让它成为调用的.cc文件中的非成员函数。
如果有别的选择时，请放在匿名空间吧。

匿名空间中添加函数