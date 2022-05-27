---
title: "C++_tips"
date: 2022-05-27T23:29:05+08:00
draft: false
tags: ["C++","language"]
category: ["C++"]
author: "clundro"
---

本篇文章主要是对阅读 [google c++ tips](https://abseil.io/tips/) 后进行的总结与记录。

## tips 1: `string_view`

将字符串传进函数，一般按如下方式。

```cpp
// C Convention
void TakesCharStar(const char* s);
// Old Standard C++ convention
void TakesString(const std::string& s);
// string_view C++ conventions
void TakesStringView(std::string_view s);     
```

其中将`std::string`转换为`const char*`需要写成`string.c_str()`。
将`const char*`转为`std::string`，直接传参，但会拷贝临时变量。而转换成`string_view`则没上述问题。

`string_view` 变量由一个`pointer`和`length`构成（这里可能有点类似切片？）

```cpp
void AlreadyHasString(const std::string& s) {
  TakesStringView(s); //O(1)
}

void AlreadyHasCharStar(const char* s) {
  TakesStringView(s); // no copy;use s.strlen()
}
```

同时`string_view`有生命周期的概念，需要确保使用时间在原字符串生命期的内部。

当想使用一个字符串数据，但不会改动时，请使用`string_view`。如果要修改，则显示转化`std::string(string_view)`。

由于`string_view`很小，因此采用`pass by value`的方式。

像`const`一样使用`string_view`，函数定义时别用`const` 限定它（参见tips 109）。

```cpp
std::cout << "Took '" << s << "'";
```
可以这样直接打印`string_view`。但由于`string_view`不一定`NUL-terminated`，因此不要用`s.data()`的写法。

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

