<a id="latent-awaiters"></a>
# Latent awaiter

[English](../Latent.md) | 简体中文

`UE5Coro::Latent` 命名空间中的这些 awaiter 与游戏线程紧密绑定，不是线程安全的。
它们必须在游戏线程上创建、使用和销毁。
持有这种局部变量的协程可以转移到其他线程，
只要它保证不会在那个线程上碰这个 latent awaiter。

该命名空间中每个函数返回的 awaiter 都满足 UE5Coro::TLatentAwaiter 概念，
这带来了独特的行为：在 latent 协程中使用时，实现会走一条间接层更少的快速路径，
并且取消会在 await 期间尽早处理，早于协程正常恢复的时刻。

在异步协程中 await 这种类型时，会在幕后创建并注册一个 latent 动作来处理该 awaiter，
这会带来额外开销。
此时会读取全局变量 GWorld，它必须在整个 co_await 期间保持有效
（在游戏线程上通常如此）。

部分 latent awaiter 是世界敏感（world sensitive）的。
它们还有一个额外要求：从调用它们的函数开始，到 co_await 结束为止，
world 必须保持不变，而不仅仅是保持有效。
这一点由 Tick 中的 ensure() 把关。

Latent 协程（相对于 latent awaiter 而言）没有这个限制，也不会访问 GWorld。
但本命名空间中的函数仍然可能访问。

### auto NextTick()
### auto Ticks(int64 Ticks)

这些函数的返回值会在经过指定数量的 tick 之后恢复协程。
不支持负值。

NextTick() 是 Ticks(1) 的便捷写法。
Ticks(0) 一开始就已完成，await 它什么也不做。

计数从调用这些函数时开始，之后再 await 它们也可以：
```cpp
// 接下来的 4 行在同一个 tick 内运行：
auto AwaiterA = Ticks(1);
auto AwaiterB = Ticks(4);
auto AwaiterC = Ticks(3);
auto AwaiterD = Ticks(2);

co_await AwaiterA; // 按要求等待 1 个 tick
co_await AwaiterB; // 1 个 tick 之后，等待 Ticks(4) 剩下的 3 个 tick
co_await AwaiterC; // 已经完成
co_await AwaiterD; // 已经完成
```

用法示例：
```cpp
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine ProcessItems(TArray<FItem> Items, FLatentActionInfo LatentInfo)
{
    // 处理元素，每个 tick 处理 128 个
    for (int i = 0; i < Items.Num(); ++i)
    {
        if (i % 128 == 0) // i==0 时这里会尽快返回到调用方
            co_await NextTick();
        ProcessItem(Items[i]);
    }
}
```

### auto Until(std::function<bool()> Function)

本函数的返回值被 co_await 时，会每个 tick 轮询一次传入的函数，
并在它第一次返回 true 时恢复协程。

它大致等价于 `while (!Function()) co_await NextTick();`，
但传入的函数是在内部的快速路径上调用的，不会反复恢复和挂起协程。

本函数假定 `Function` 不是世界敏感的。
如果它用到了 GWorld，请确保它能应对 world 的变化，或者阻止 world 变化。

示例：
```cpp
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine Example(FLatentActionInfo LatentInfo)
{
    co_await Until([&] { return bProceedWithExample; });
    Done();
}
```

### auto Seconds(double Seconds)
### auto UnpausedSeconds(double Seconds)
### auto RealSeconds(double Seconds)
### auto AudioSeconds(double Seconds)

这些函数返回的 awaiter 会等待指定的一段时间后再恢复协程，
时间以调用这些函数时的当前 world（GWorld）来衡量。
因此，时间膨胀和/或暂停可能会影响它们，并且同一个 tick 内的一切都视为同时发生。

|        |Seconds|UnpausedSeconds|RealSeconds|AudioSeconds|
|--------|:-----:|:-------------:|:---------:|:----------:|
|时间膨胀|✅      |✅              |❌          |❌           |
|暂停    |✅      |❌              |❌          |✅           |

✅=遵守，❌=忽略

示例：
```cpp
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine CountDown(int Value, FLatentActionInfo LatentInfo)
{
    for (int i = Value; i > 0; --i)
    {
        UE_LOGFMT(LogTemp, Display, "{0}...", i);
        co_await Seconds(1.0);
    }
    UE_LOGFMT(LogTemp, Display, "Time's up!");
}
```

等待负的时长会触发 `ensure`，并立即结束。

这些函数返回的是世界敏感的 awaiter。
如果需要一个线程安全、不需要 world 的 RealSeconds 替代品，
请参见 UE5Coro::Async::PlatformSeconds。

### auto UntilTime(double Seconds)
### auto UntilUnpausedTime(double Seconds)
### auto UntilRealTime(double Seconds)
### auto UntilAudioTime(double Seconds)

它们的行为与对应的 Seconds 版本完全相同，但返回值等待的是当前 world
到达指定的时间点，而不是经过一段时长。
等待一个过去的时间点同样会触发 `ensure` 并立即结束。

例如，`UntilTime(GWorld->GetTimeSeconds() + 10)` 等价于 `Seconds(10)`。

更多细节请参见本节正上方的 Seconds 系列函数。

这些函数返回的是世界敏感的 awaiter。
UntilRealTime 对应的异步版本是 UE5Coro::Async::UntilPlatformTime。

### auto SecondsForActor(AActor* Actor, double Seconds)
### auto UnpausedSecondsForActor(AActor* Actor, double Seconds)

它们的行为分别与 Seconds 和 UnpausedSeconds 类似，
但还会把 actor 自身的时间膨胀（custom time dilation）考虑在内。

如果 actor 仍然存活并且经过了指定的时长，await 表达式的结果为 true；
如果 actor 被销毁、延时因此被提前截断，结果为 false。

这些函数返回的是世界敏感的 awaiter。

示例：
```cpp
using namespace UE5Coro::Latent;

FVoidCoroutine Example(AActor* Actor, FLatentActionInfo LatentInfo)
{
    bool bCompleted = co_await SecondsForActor(Actor, 1);
}
```

### auto WhenAny(TLatentContext\<const UObject\> LatentContext, TAwaitable auto&&... Awaitables)

文档见[这里](Aggregate.md#auto-latentwhenanytlatentcontextconst-uobject-latentcontext-tawaitable-auto-awaitables)。

### auto WhenAll(TLatentContext\<const UObject\> LatentContext, TAwaitable auto&&... Awaitables)

文档见[这里](Aggregate.md#auto-latentwhenalltlatentcontextconst-uobject-latentcontext-tawaitable-auto-awaitables)。
