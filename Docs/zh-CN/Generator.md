# 生成器

[English](../Generator.md) | 简体中文

> [!NOTE]
> 以 C++23 或更高版本为目标时，TGenerator 已被弃用。<br>
> 推荐改用 std::generator。<br>
> 定义宏 `UE5CORO_DISABLE_GENERATOR_DEPRECATION` 可以继续使用它。

函数返回 TGenerator\<T\>，就可以通过它产出（yield）任意数量的值（包括无限个），
而由调用方控制何时取、取多少。
每次取值时，函数都会恢复执行，一直运行到下一个 `co_yield` 或运行结束。

与为 TArray 分配内存相比，这样做可能更直接，也更节省内存，
因为值是按需逐个生成和返回的。
生成器还能受益于编译器的
[HALO](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1365r0.pdf) 优化。

默认构造的 TGenerator 不会产出任何元素，就好像这个生成器被实现成了
`{ co_return; }`（但效率更高）。<br>
移动一个生成器，会在两个对象之间转移协程的执行状态，而不影响协程本身。
被移走的 TGenerator 会变得与默认构造的完全相同。

使用 TGenerator 有多种方式。
它提供了直接操作的方法，还提供了一个只有 1 个指针大小、效率极高的迭代器。
这个迭代器可以用于 STL 风格、LLVM 风格和 Unreal 风格的循环：
begin() 与 CreateIterator() 完全相同。

迭代器类型的规范名称是 `TGenerator<T>::iterator`。
它在 `UE5Coro::Private` 命名空间中的内部名称不应出现在代码里，
因为它随时可能改变，且不会事先标记弃用。
如果这种情况发生，`iterator` 会随之更新为指向新类型。

## 手动 API

TGenerator 本身有许多方法，可以直接与它所代表的函数的执行过程交互。

下面这些函数可以这样配合使用：
```cpp
for (TGenerator<int> Gen = Example(); Gen; Gen.Resume())
    UE_LOGFMT(LogTemp, Display, "Current value = {0}", Gen.Current());
```

### explicit TGenerator\<T\>::operator bool() const noexcept

只要 Current 中有值，生成器就会转换为 true；
底层函数调用结束后，它变为 false。

> [!NOTE]
> 与基于 IEnumerator 的 C# 协程不同，通过 TGenerator 无法观察到
> “第一次 yield 之前”的状态。
>
> C++ 使用的是“最后一个之后”的状态，例如 end() 所代表的那种状态。

### bool TGenerator\<T\>::Resume()

让生成器运行一步。
如果生成器产出了值，返回 true；如果已经结束，返回 false。
最后产出的值可以通过 Current() 读取。
调用此方法会使所有活跃的迭代器失效。

恢复一个已经结束的生成器是安全的：它什么也不做，并且一直返回 false。

### T& TGenerator\<T\>::Current() const

返回生成器当前正在 co_yield 的值的引用。
对已经结束的生成器调用 Current() 是未定义行为。

此时生成器正停在 co_yield 表达式内部，所以即使是临时对象，
也可以作为左值引用访问：

```cpp
using namespace UE5Coro;

TGenerator<FString> Example()
{
    co_yield TEXT("Hello!"); // 这里的 FString 是一个临时对象……
}

// ……但只要函数还冻结在 co_yield 中间，这个 FString 就一直存活，
// 因此它在这里的生命周期足够长：
TGenerator<FString> Generator = Example();
if (Generator)
{
    FString& String = Generator.Current();
    SomethingThatTakesFStringRef(String);
    // 依然存活！
}
// 这次调用会使引用失效
Generator.Resume();
```

如有需要，可以把这些值从生成器中移出，前提是最多只有一个线程、最多移出一次。
值会直接从 co_yield 表达式中移出。
为了最高效率，TGenerator 不会为这个值做任何复制，也不提供额外的存储。

## STL 风格的迭代器 API

提供了常见的 begin() 和 end() 这一对函数，std::begin 和 std::end 会找到它们。
它们可以单独使用，也可以通过基于范围的 for 循环使用：

```cpp
for (auto& Value : SomeGenerator())
    DoSomethingWith(Value);
```

这种写法（生成器完全存在于一个基于范围的 for 循环中）是编译器优化的最佳场景，
推荐作为使用 TGenerator 的主要方式。

### iterator TGenerator\<T\>::begin() noexcept

返回一个代表生成器**当前（！）**状态的迭代器。
没有倒回功能。
此函数的行为与 CreateIterator() 完全相同。

用 ++ 移动到生成器末尾的迭代器会等于 end()。
对等于 end() 的迭代器使用前缀 * 或 -> 是未定义行为。

允许复制迭代器，但试图通过多个迭代器操作同一个生成器是未定义行为。
一旦进行了复制，或者再次调用了 begin() 或 CreateIterator()，
其他所有迭代器都应视为失效。
同样，对一个迭代器调用 ++ 会直接操作生成器，并使其他所有活跃的迭代器失效。

出于传统提供了后缀 ++ 运算符，但它返回 void，行为与前缀自增相同。
不可能返回生成器恢复之前的状态副本。

### iterator TGenerator\<T\>::end() const noexcept

返回一个哨兵迭代器，它与已经走过生成器末尾的迭代器比较相等。
这些迭代器不会因生成器状态的改变而失效，可以随意复制。

因此也支持 LLVM 风格的 for 循环（但没有必要）：
```cpp
auto Generator = SomeGenerator();
for (auto i = Generator.begin(), e = Generator.end(); i != e; ++i)
    DoSomethingWith(*i);
```

## Unreal 风格的迭代器 API

作为一个 Unreal 插件，不支持 Unreal 风格的迭代就说不过去了。

### iterator TGenerator\<T\>::CreateIterator() noexcept

返回一个代表生成器**当前（！）**状态的迭代器。
没有倒回功能。
此函数的行为与 begin() 完全相同。

该迭代器可以转换为 bool，并且有 ++ 运算符，以支持 Unreal 风格的迭代：

```cpp
for (auto It = SomeGenerator.CreateIterator(); It; ++It)
    DoSomethingWith(*It);
```

关于用法，STL 风格迭代器 API 的那些说明同样适用。
转换为 false 的 UE 风格迭代器等价于等于 end() 的迭代器，
表示背后的 TGenerator 已经结束，Current() 中不再有值。
