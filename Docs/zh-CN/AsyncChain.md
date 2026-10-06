# 异步链

[English](../AsyncChain.md) | 简体中文

UE5Coro::Async::Chain 用来调用那些不认识协程、但接受一个委托参数的函数，
并在该委托被触发时恢复协程。

被链接函数的参数以及委托的参数都必须是 _MoveConstructible_（可移动构造）的。
委托的返回类型必须是 _DefaultConstructible_（可默认构造）的，或者是 void。

Async::Chain 主要适用于这样的引擎函数：它们会复制一份预先绑定好的委托，
之后对原委托的任何改动都不再理会。
如果函数只是引用委托，并允许调用方之后再绑定或解绑，那么委托可以
[直接 co_await](Implicit.md#delegates)。

> [!CAUTION]
> 不要链接那些可能调用委托零次或多次的函数。
> 第一次之后再到来的回调会引发未定义行为，除非委托是 DYNAMIC 的，
> 或者被链接的函数保证会遵守对传给它的原委托所做的 Unbind/Remove。
>
> 对于 Async::Chain 不支持的所有情况，或者对它是否合适、是否安全存疑时，
> 可以改用安全的[通用变通方案](Implicit.md#generic-workarounds)。

只要可能，就应该优先直接 `co_await`，而不是用 Async::Chain。
Async::Chain 会在恢复协程之前干净地取消对其临时委托的订阅，
但它假定被链接的函数会无视这一点。
它**不**支持加速取消，以确保引擎手里那份已绑定委托的副本始终有东西可调用，
即使协程已经被取消。
除非取消被阻止，否则取消会在委托被触发后立即处理。

### auto Chain(? (*Function)(?...), A&&... Args)
### auto Chain(T* Object, ? (T::*Function)(?...), A&&... Args)

完整的、<!--更加-->晦涩难懂的声明在文档中略去。
第一个重载用于指向静态函数的指针，第二个用于类的非静态成员函数。

被链接的函数必须恰好接受一个委托参数。
支持的委托类型列表见[此页](Implicit.md#delegates)。
委托参数可以是值、引用或指针。

插件会创建并绑定一个合适的委托传给该函数，函数的其余参数则由 `Args` 转发而来。
委托会直接、同步地调用进协程。

await 表达式的结果是一个未指明的类型，可以配合结构化绑定，按需接收委托的参数。
引用参数会以引用形式传递，并且可以写入。
这些写入会通过委托调用传回原先被引用的变量。

如果协程不想接收参数，可以放心地丢弃 await 表达式的值。
不支持以结构化绑定声明之外的任何方式使用 await 表达式的返回值
（包括把它整个存进一个 `auto` 局部变量）。

引用和指针的有效性取决于委托的调用方，但即使是生命周期最短的引用，
也会在下一次 co_await 或 co_return 之前保持有效。

被链接函数的同步返回值会被丢弃。
对于返回类型不是 void 的委托，会在协程下一次 co_await 或 co_return 时，
向调用方返回一个默认构造的值。
即使协程 co_return 了另一个结果，委托收到的也是这个默认构造的值。
委托的返回类型与协程的结果类型彼此独立。

示例：
```c++
using namespace UE5Coro;

// DECLARE_DYNAMIC_MULTICAST_DELEGATE(FExampleDelegate);
// void ChainMe(FExampleDelegate*);
// void AActress::ChainThis(int, const TDelegate<void(FString)>&, float);

TCoroutine<> AActress::Example()
{
    // 跳过委托参数，为其余参数提供实参：
    co_await Async::Chain(&ChainMe);
    auto&& [String] = co_await Async::Chain(this, &AActress::ChainThis, 1, 2.0f);
}
```
