---
title: "C++_tips"
date: 2022-05-27T23:29:05+08:00
draft: false
tags: ["C++","language"]
category: ["C++"]
author: "clundro"
---

本篇文章主要是对阅读 [google c++ tips](https://abseil.io/tips/) 后进行的盲目记录与总结。

## tips 1: `string_view`

----

将字符串传进函数，一般按如下方式。

```cpp
// C Convention
void TakesCharStar(const char* s);
// Old Standard C++ convention
void TakesString(const std::string& s);
// string_view C++ conventions, from c++ 17
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

同时`string_view`本身不拥有数据，因此有生命周期的概念，需要确保使用时间在原字符串生命期的内部。

当想使用一个字符串数据，但不会改动时，请使用`string_view`。如果要修改，则显示转化`std::string(string_view)`。

由于`string_view`很小，因此采用`pass by value`的方式。

像`const`一样使用`string_view`，函数定义时别用`const` 限定它（参见tips 109）。

```cpp
std::cout << "Took '" << s << "'";
```

可以这样直接打印`string_view`。但由于`string_view`不一定`NUL-terminated`，因此不要用`s.data()`。


## tips 3: StrCat() & StrAppend()

---

`string concat`即`std::string::operator+`是低效的。
其中

```cpp
std::string foo = LongStr1(),bar = LongStr2(),baz = LongStr3();
string foobar1 = foo + bar + baz; // 1
std::string foobar2 = absl::StrCat(foo, bar, baz);  // 2
```

在两参数情况下，方式1和2效率相同。但由于没有进行三参数的重载，方式1会像下面处理。

```cpp
std::string temp = foo + bar;
std::string foobar = std::move(temp)+baz;
```

注意到一点，`std::move(temp)+baz`自c++11开始等价于`std::move(temp.append(baz))`。因此可能出现一种情况，分配给`temp`的初始buffer不够囊括下`foobar`,则会导致新增的`reallocation`和`copy`，因此最坏情况`n`长度的`string`需要`O(n)`重分配时间。

`absl::StrCat`位于[absl/strings/str_cat.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_cat.h)。之后有机会好好介绍下函数的实现方式，这里简要介绍下。

```cpp
namespace absl{
namespace strings_internal {
template <size_t max_size>
struct AlphaNumBuffer {
  std::array<char, max_size> data;
  size_t size;
  };
}
class AlphaNum{
 private:
  absl::string_view piece_;
  char digits_[numbers_internal::kFastToBufferSize];
};

} 
```

通过设置一个固定大小的数组来存储`internal`,之后用`AlphaNum`作为`StrCat()`和`StrAppend()`包装的类，用`piece_`和`digits_`来存储`strings_internal`的信息。参看.cc文件的实现，对于方式2实现的原理大概就是将所有`string`转换成`AlphaNum`,然后提前计算下所需要的`size`并创建`result`字符串，通过指向`result`末端的指针一一`memcpy`。总共需要一次预分配内存和多次局部拷贝。

```cpp
foobar += foo + bar + baz; // 1
absl::StrAppend(&foobar, foo, bar, baz); //2
```

类似的原理实现`absl::StrAppend`。这俩函数支持`int32_t, uint32_t, int64_t, uint64_t, float, double, const char*, string_view`这些类型的转化。当然可以看出，节约的时间在于临时量的拷贝和多次重分配。

## tips 5:消失的艺术

```cpp
const char* p1 = (s1 + s2).c_str();             // Avoid!
const char* p2 = absl::StrCat(s1, s2).c_str();  // Avoid!
```

两种写法产生临时变量，通过`c_str()`拿到指针。但根据c++ 17标准，

>Temporary objects are destroyed as the last step in evaluating the full-expression that (lexically) contains the point where they were created.” (A “full-expression” is “an expression that is not a subexpression of another expression”

因此当语句的赋值结束后，临时变量销毁，生命周期的限制便会产生`dangling ptr`的问题。

### option 1: 持有

不如持有临时变量。临时变量在栈上创建，经过`rvo`(临时变量的`move`语义)，直接构建，而不需要拷贝临时变量。

```cpp
std::string tmp_1 = s1 + s2;
std::string tmp_2 = absl::StrCat(s1, s2);
```

### option 2: 用引用指向临时变量

根据c++ 17 标准，

>The temporary to which the reference is bound or the temporary that is the complete object of a sub-object to which the reference is bound persists for the lifetime of the reference.

用引用持有临时变量，并不会比 `option 1`更优，一般情况安全。但遇到`Exception`时，会有`dangling reference`的风险。

```cpp
const std::string& tmp_1 = s1 + s2;
const std::string& tmp_2 = absl::StrCat(s1, s2);

// If the compiler can see you’re storing a reference to a
// temporary object’s internals, it will keep the whole
// temporary object alive.

//GeneratePerson() returns an object; GeneratePerson().name
// is clearly a sub-object:
const std::string& person_name = GeneratePerson().name; // safe

//GenerateDiceRoll() returns an object; the compiler can’t tell
// if GenerateDiceRoll().nickname() is a sub-object.
const std::string& nickname = GenerateDiceRoll().nickname(); // BAD!
```

情况取决于编译器是否知晓临时变量内部值的引用需要维持。
当然还有一种方式，不要返回对象。

## tips 10: 精简地拆分字符串

通常拆分字符串的函数会因为各种输入参数，输出参数和语义需求而弄出多个版本.
谷歌于是造了一个统一的`absl::StrSplit()`函数,代码位于[absl/strings/str_split.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_split.h).

```cpp
// Splits on commas. Stores in vector of string_view (no copies).
std::vector<absl::string_view> v = absl::StrSplit("a,b,c", ',');

// Splits on commas. Stores in vector of string (data copied once).
std::vector<std::string> v = absl::StrSplit("a,b,c", ',');

// Splits on literal string "=>" (not either of "=" or ">")
std::vector<absl::string_view> v = absl::StrSplit("a=>b=>c", "=>");

// Splits on any of the given characters (',' or ';')
using absl::ByAnyChar;
std::vector<std::string> v = absl::StrSplit("a,b;c", ByAnyChar(",;"));

// Stores in various containers (also works w/ absl::string_view)
std::set<std::string> s = absl::StrSplit("a,b,c", ',');
std::multiset<std::string> s = absl::StrSplit("a,b,c", ',');
std::list<std::string> li = absl::StrSplit("a,b,c", ',');

// Equiv. to the mythical SplitStringViewToDequeOfStringAllowEmpty()
std::deque<std::string> d = absl::StrSplit("a,b,c", ',');

// Yields "a"->"1", "b"->"2", "c"->"3"
std::map<std::string, std::string> m = absl::StrSplit("a,1,b,2,c,3", ',');
```

这里简要介绍(还没时间看原理),通过`string_view`指向输入的字符串,然后分割出来的`string_view`根据返回值类型,决定是否拷贝.因此由于底层实现使用`string_view`，避免了拷贝，所以比较高效。

## tips 10: 返回值策略

`RVO(return value optimization)`是被大部分编译器实现的`feature`。

```cpp
static SomeBigObject SomeBigObjectFactory(...) {
  SomeBigObject local;
  ...
  return local;
}
SomeBigObject obj = SomeBigObject::SomeBigObjectFactory(...);
```

由于`RVO`技术，`compiler`将调用者`obj`的地址直接传递给被调用者`SomeBigObjectFactory`。

那么什么时候`compiler`不会使用`RVO`技术呢？

```cpp
// RVO won’t happen here; 调用  SomeBigObject& operator=(const SomeBigObject& s);
obj = SomeBigObject::SomeBigObjectFactory(s2);
// obj has been defined;
```

如果调用者重新使用一个值来存储返回值，则不会进行`RVO`。当然这种情况下，会在`move-enabled`类型内调用移动语义。

```cpp
// RVO won’t happen here:
static SomeBigObject NonRvoFactory(...) {
  SomeBigObject object1, object2;
  object1.DoSomethingWith(...);
  object2.DoSomethingWith(...);
  if (flag) {
    return object1;
  } else {
    return object2;
  }
}

// RVO will happen here:
SomeBigObject local;
if (...) {
  local.DoSomethingWith(...);
  return local;
} else {
  local.DoSomethingWith(...);
  return local;
}
```

如果被调用者返回多个变量作为返回值，也不会做`RVO`。如果是使用一个变量但返回多个地方，则会做`RVO`。

### temporaries

此外，`RVO`不仅命名变量上生效，同样也在临时对象上生效，即调用者返回临时变量的情况。

```cpp
// RVO works here:
SomeBigObject SomeBigObject::ReturnsTempFactory(...) {
  return SomeBigObject::SomeBigObjectFactory(...);
}

// RVO works here:
EXPECT_EQ(SomeBigObject::SomeBigObjectFactory(...).Name(), s);
```

当调用者立刻使用返回值(被存储在临时对象中)时，`RVO`也会生效。

记住一句话，对代码需要`copy`，就会执行`copy`，不管`copy`会不会优化。不要为了高效而牺牲正确性。

## tips 24: 拷贝

当代码为同一个数据出现两个名字时，那就需要一份拷贝。
如果你避免引入新名字,那么编译器可能会帮你去除掉拷贝。

```cpp
std::string build();

std::string foo(std::string arg) {
  return arg;  // no copying here, only one name for the data “arg”.
}

void bar() {
  std::string local = build();  // only 1 instance -- only 1 name

  // no copying, a reference won’t incur a copy
  std::string& local_ref = local;

  // one copy operation, there are now two named collections of data.
  std::string second = foo(local);
}
```

记住一句话

> everything you learned about copies in C++ a decade ago is wrong.

阅读c++ 白皮书的时候也发现，历史遗留问题改动还不小。


## tips 36: New Join API

了解下 `absl::StrJoin` 吧。它支持`std::string, absl::string_view, int, double – any type that absl::StrCat() supports`。

```cpp
std::vector<std::string> v = {"a", "b", "c"};
std::string s = absl::StrJoin(v, "-");
// s == "a-b-c"

std::vector<absl::string_view> v = {"a", "b", "c"};
std::string s = absl::StrJoin(v.begin(), v.end(), "-");
// s == "a-b-c"

std::vector<int> v = {1, 2, 3};
std::string s = absl::StrJoin(v, "-");
// s == "1-2-3"

const int a[] = {1, 2, 3};
std::string s = absl::StrJoin(a, "-");
// s == "1-2-3"
```

如果你想`join`一个`StrCat`不支持的类型，就添加一个自定义`Formatter`。

```cpp
std::map<std::string, int> m = {{"a", 1}, {"b", 2}, {"c", 3}};
std::string s = absl::StrJoin(m, ";", absl::PairFormatter("="));

std::vector<Foo> foos = GetFoos();
// use lambda as formatter
std::string s = absl::StrJoin(foos, ", ", [](std::string* out, const Foo& foo) {
  absl::StrAppend(out, foo.ToString());
});
```

源码在[absl/strings/str_join.h](https://github.com/abseil/abseil-cpp/blob/master/absl/strings/str_join.h)。简而言之，就是对 tips 3中提到的`strings_internal`传递`$1`参数和`Formatter`做`join`计算。最后利用`strings_internal`拷贝到`string`。

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

