---
title: "C++_tips"
date: 2022-05-27T23:29:05+08:00
draft: false
tags: ["language"]
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

**这里简要介绍,通过`string_view`指向输入的字符串,然后分割出来的`string_view`根据返回值类型,决定是否拷贝.因此由于底层实现使用`string_view`，避免了拷贝，所以比较高效。**

## tips 11: 返回值策略

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
// obj was defined;
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

记住一句话，当代码需要时`copy`，就会执行`copy`，不管`copy`会不会优化。（意思是，这些地方依然需要`copy`语义，只是被`RVO`进行`copy`优化了 ）不要为了高效而牺牲正确性。

**简单点，直接在局部函数中返回临时变量。**

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

**简而言之，涉及到`string`操作的建议用`absl::string`**

## tips 42:最好用工厂函数初始化方法

如果在当前环境禁用`exception`，则c++ `ctor`必须成功，毕竟没有通知`caller`构造失败的方法了。如果你使用`abort`，则会使整个程序崩溃，对于产品代码得不偿失。

有一种简单的方式是提供`factory function`来创建和初始化`instance`，并返回它的指针或者`absl::optional`(Tips 123)，用`null`表示失败(option的做法有类型统一的好处)。

```cpp
// foo.h
class Foo {
 public:
  // Factory method: creates and returns a Foo.
  // May return null on failure.
  static std::unique_ptr<Foo> Create();

  // Foo is not copyable.
  Foo(const Foo&) = delete;
  Foo& operator=(const Foo&) = delete;

 private:
  // Clients can't invoke the constructor directly.
  Foo();
};

// foo.c
std::unique_ptr<Foo> Foo::Create() {
  // Note that since Foo's constructor is private, we have to use new.
  return absl::WrapUnique(new Foo());
}
```

`Foo::Create()`只会暴露出成功初始化的对象，同时也能像初始化方法表达失败。工厂函数的另一个优点是它能返回`instances of any subclass of the return type`（使用`absl::optional`作为返回类型当然就不行了）。这允许你使用不同的实现时，而不需要更新用户代码。甚至根据用户输入，动态选择实现类。

**这里说的做法大概是根据`Create()`函数的输入参数进行选择`impl`，返回参数的类型如果是指针的话，子类型也可以返回。**

该方法的缺点是生成的是分配在堆上的对象，对`value-like`类在栈上工作不友好。当`derived class ctor`需要初始化`base class`时，工厂函数不能使用，因此初始化方法在基类的`protected API`中是必要的。

**这里我所理解的工厂函数，是通过某个函数包装原函数的指针。该指针同时包含原函数是否执行成功的状态信息。**

## tips 45: 库代码中避免 `Flags`

在产品代码中`flags`的通常使用，尤其是在库代码中，是一个巨大的错误。

`Flags`是全局变量时，只会让事情更糟。无法阅读代码知道变量的值，不知道多次版本更迭后`flag`值是否保持不变。

谨慎使用`flag`。使用数字`flag`可以考虑变成`compile-time constants`。

**这里说的`flag`大概是一些宏定义或者用于标记性质的常量。**

## tips 49:参数依赖的查找(Argument-Dependent Lookup)_unfinished

一个函数调用表达式，类似`func(a,b,c)`，没有`::`域名操作符时，称为非限定的（`unqualified`），此时编译器会进行匹配函数声明的查找。
>the set of search scopes is augmented by namespaces associated with the function argument types. This additional lookup is called Argument-Dependent Lookup (ADL).

## tips 55: 命名计数和 `unique_ptr`

通俗说，一个值的 `name` 表示任何值类型的变量(不是指针，也不是引用)，存在在任何作用域且持有某个特别的数据值。（对于专门的C++律师，我们说`name`一般指的是`lvalue`）由于`unique_ptr`特殊行为需求，我们需要确保任何`unique_ptr`持有的值都只有一个名字。

需要注意的是，C++ 语言委员会为 std::unique_ptr 选择了一个非常恰当的名称。任何存储在`unique_ptr`的非空指针值 在任何时候都只能出现在一个`unique_ptr`中。标准库的设计符合这个要求。

在每一行，计算在该点（无论是否在范围内）活动的名称的数量，这些名称引用包含相同指针的 std::unique_ptr。 如果您发现同一指针值属于多个名称的任何行，那就是错误！

```cpp
std::unique_ptr<Foo> NewFoo() {
  return std::unique_ptr<Foo>(new Foo(1));
}

void AcceptFoo(std::unique_ptr<Foo> f) { f->PrintDebugString(); }

void Simple() {
  AcceptFoo(NewFoo());
}

void DoesNotBuild() {
  std::unique_ptr<Foo> g = NewFoo();
  AcceptFoo(g); // DOES NOT COMPILE!
}

void SmarterThanTheCompilerButNot() {
  Foo* j = new Foo(2);
  // Compiles, BUT VIOLATES THE RULE and will double-delete at runtime.
  std::unique_ptr<Foo> k(j);
  std::unique_ptr<Foo> l(j);
}
```

在 `Simple()` 中，用 `NewFoo()` 分配的唯一指针只有一个可以引用它的名称：`AcceptFoo()` 中的名称“f”。

将其与 `DoesNotBuild()` 进行对比：使用 `NewFoo()` 分配的唯一指针有两个引用它的名称：`DoesNotBuild()` 的“g”和 `AcceptFoo()` 的“f”。

这就是常见的唯一性违规。在执行的任何给定点，std::unique_ptr 持有的任何值（或更一般地，任何仅移动类型）都只能由一个不同的名称引用。 任何看起来像引入附加名称的副本都是禁止的，并且不会编译：

```cpp
scratch.cc: error: call to deleted constructor of std::unique_ptr<Foo>'
  AcceptFoo(g);
```

即使编译器没有捕捉到你，std::unique_ptr 的运行时行为也会。 任何时候你“超越”编译器（参见 `SmarterThanTheCompilerButNot()`）并引入多个 std::unique_ptr 名称，它可能会编译（目前），但你会遇到运行时内存问题。

那么问题来了：**我们如何删除一个名字？** C++11 也为此提供了解决方案，形式为 std::move()。

```cpp
 void EraseTheName() {
   std::unique_ptr<Foo> h = NewFoo();
   AcceptFoo(std::move(h)); // Fixes DoesNotBuild with std::move
}
```

对 std::move() 的调用实际上是一个名称擦除器：从概念上讲，您可以停止将“h”计算为指针值的名称。 这现在通过了 distinct-names 规则：在分配给 `NewFoo()` 的唯一指针上有一个名称（“h”），并且在对 `AcceptFoo()` 的调用中再次只有一个名称（“f”）。 通过使用 std::move()，我们保证在为它分配新值之前不会再次读取“h”。

**简而言之， 多一个名字便多一分拷贝。`unique_ptr`只能用一个名字，如果想更换名字，就用`move`**

## tips 163: 传递 `absl::optional` 参数

c++ 17已经引入`optional`了。

遇到个问题，我们需要一个函数能接受可能存在也可能不存在的参数。那么这时候可能使用`absl::optional`。但如果这个对象足够大以至于我们需要传递引用，那`absl::optional`就不好使了。

```cpp
void MyFunc(const absl::optional<Foo>& foo);  // Copies by value
void MyFunc(absl::optional<const Foo&> foo);  // Doesn't compile
```

如上所示，第一个选项可能无法满足您的要求。 如果有人将 Foo 传递给 MyFunc，Foo 将按值复制到 `absl::optional<Foo>`，然后将通过引用传递给函数。 如果您的目标是避免复制 Foo，那么您没有。

第二个选项会很棒，但不幸的是 absl::optional 不支持。（这篇文章2020-4-6更新的，不知道现在如何）

这时候，我们可以传递`const *`用`nullptr`代表不存在。

```cpp
void MyFunc(const Foo* foo);
```

这样和`const Foo&`传递一样高效，且支持空值。

std::optional 的文档指出，您可以使用 std::reference_wrapper 来解决不支持可选引用的事实：

```cpp
void MyFunc(absl::optional<std::reference_wrapper<const Foo>> foo);
```

类似这样，但这太长且不易阅读，因此我们不推荐。

因此**总结一下，如果您拥有可选的东西，则可以使用 absl::optional 。 例如，类成员和函数返回值通常适用于 absl::optional。 如果您不拥有可选的东西(即存在空的情况)，只需使用指针，如上所述。**

异常的问题，如果您的对象足够小以至于不需要通过引用，您可以将对象包装在 absl::optional 中,例如

```cpp
void MyFunc(absl::optional<int> bar);
```

如果希望你的函数的所有调用者已经在 absl::optional 中有一个对象，那么你可以使用 const absl::optional&。 但是，这种情况很少见； 它通常仅在您的函数在您自己的文件/库中是私有的时才会发生。

## tips 166: 什么时候 `Copy is not a Copy`

从c++ 17 开始，对象可能被原地创建。

```cpp
class BigExpensiveThing {
 public:
  static BigExpensiveThing Make() {
    // ...
    return BigExpensiveThing();
  }
  // ...
 private:
  BigExpensiveThing();
  std::array<OtherThing, 12345> data_;
};

BigExpensiveThing MakeAThing() {
  return BigExpensiveThing::Make();
}

void UseTheThing() {
  BigExpensiveThing thing = MakeAThing();
  // ...
}
```

在 C++17 之前，上面拷贝或移动对象的次数最多为三个：每个 return 语句一个，初始化事物时还有一个。 这是有道理的：每个函数都可能将 BigExpensiveThing 放在不同的位置，因此可能需要移动以将值放在最终调用者想要的位置。 然而，在实践中，对象总是在变量 thing 中“就地”构建，不执行任何移动，并且 C++ 语言规则允许“省略”这些移动操作以促进这种优化。

在 C++17 中，保证此代码执行零复制或移动。 事实上，即使 BigExpensiveThing 不可移动，上面的代码也是有效的。 BigExpensiveThing::Make 中的构造函数调用直接构造了UseTheThing 中的局部变量`thing`。

编译器看到`BigExpensiveThing()`时，并不会立即创建临时变量。
相反，它将该表达式视为有关如何初始化某些最终对象的指令，但会尽可能长时间地推迟创建（正式地，“物化”）临时对象。

通常，对象的创建会延迟到对象被命名。 命名对象（上例中的 `thing`）使用通过评估初始化程序找到的指令直接初始化。 如果名称是引用，则将物化一个临时对象来保存该值。

因此，对象直接在正确的位置构造，而不是在其他地方构造然后复制。 这种行为有时被称为“保证复制省略”(`guaranteed copy elision`)，但这是不准确的：一开始就没有副本。

**简而言之，对象在首次命名之前不会被复制。通过值返回没有额外开销。**

(并且根据tips 11,即使在给定名称之后，由于nrvo，局部变量在从函数返回时仍可能不会被复制)

### 那么有一个问题，什么时候未命名对象被拷贝呢？

在两种 corner case下，使用未命名对象无论如何都会导致副本

构造基类：在构造函数的基类初始值设定项列表中，即使从基类类型的未命名表达式构造时也会进行复制。 这是因为类在用作基类时可能会有一些不同的布局和表示（由于 virtual base classes和 vpointer 值），因此直接初始化基类可能不会导致正确的表示

```cpp
class DerivedThing : public BigExpensiveThing {
 public:
  DerivedThing() : BigExpensiveThing(MakeAThing()) {}  // might copy data_
};
```

传递或返回小的平凡对象(trivial objects)：当一个足够小的可平凡复制的对象被传递给函数或从函数返回时，它可能会在寄存器中传递，因此在传递之前和之后可能有不同的地址。

```cpp
struct Strange {
  int n;
  int *p = &n;
};
void f(Strange s) {
  CHECK(s.p == &s.n);  // might fail
}
void g() { f(Strange{0}); }
```

### 还有一个细节，什么是  Value Category （值的范畴）

在C++中有两种表述。

1. 产生值的那些词，例如 1 或 MakeAThing() - 您可能认为具有非引用类型的表达式。

2. 那些产生一些现有对象的位置的词，例如 s 或 thing.data_[5] - 您可能认为具有引用类型的表达式。

这种划分为`value category`。前者是`prvalue`,后者是`glvalue`。我们前面所说的未命名对象即为`prvalue`.

所有纯右值表达式都在确定它们将值放在哪里的上下文中进行评估，并且纯右值表达式的执行用它的值初始化那个位置。

```cpp
 BigExpensiveThing thing = MakeAThing();
```

prvalue 表达式 MakeAThing() 被评估为`thing`变量的初始化程序，因此 MakeAThing() 将直接初始化`thing`。 构造函数将指向`thing`的指针传递给 MakeAThing()，并且 MakeAThing() 中的 return 语句初始化指针指向的内容。

```Cpp
return BigExpensiveThing();
```

相似的，编译器有一个指向要初始化的对象的指针，并通过调用 BigExpensiveThing 构造函数直接初始化该对象。

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
