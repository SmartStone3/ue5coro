# Latent 时间轴

[English](../LatentTimeline.md) | 简体中文

这四个内置协程以 latent 模式运行，因此只能在游戏线程上调用；
如果它们的 world context 对象被销毁，它们会自我取消。

TCoroutine\<\> 本身并不直接满足 TLatentAwaiter 概念，
但在 latent 协程中 await 它时，它会提供 latent 行为。
（在幕后创建的那个临时 awaiter 确实满足该概念。）

### TCoroutine<> Timeline(const UObject* WorldContextObject, double From, double To, double Duration, std::function<void(double)> Update, bool bRunWhenPaused = false)
### TCoroutine<> UnpausedTimeline(const UObject* WorldContextObject, double From, double To, double Duration, std::function<void(double)> Update, bool bRunWhenPaused = true)
### TCoroutine<> RealTimeline(const UObject* WorldContextObject, double From, double To, double Duration, std::function<void(double)> Update, bool bRunWhenPaused = true)
### TCoroutine<> AudioTimeline(const UObject* WorldContextObject, double From, double To, double Duration, std::function<void(double)> Update, bool bRunWhenPaused = false)

这四个函数会在游戏线程上每个 tick 反复调用传入的回调，
传入一个在 `From` 和 `To` 之间线性插值的值（两端都包含）。
`From` 可以大于 `To`，以实现反向插值。

NaN、无穷大等值不做处理，但会遵守引擎宏 `ENABLE_NAN_DIAGNOSTIC`：
定义了它时，这些协程会与引擎配合，检测这些特殊值。
这会带来一些额外开销。

这几个函数的实现几乎相同，区别在于它们使用 UWorld 的哪种时间，
以及 `bRunWhenPaused` 的默认值：

|                     |Timeline|UnpausedTimeline|RealTimeline|AudioTimeline|
|---------------------|:------:|:--------------:|:----------:|:-----------:|
|时间膨胀             |✅       |✅               |❌           |❌            |
|暂停                 |✅       |❌               |❌           |✅            |
|bRunWhenPaused 默认值|`false` |`true`          |`true`      |`false`      |

✅=遵守，❌=忽略

在遵守暂停的时间轴上于暂停期间运行，会一直重复同一个值，直到 world 取消暂停。

下面的示例展示了它如何融入一个更大的协程
（不过冷却之类的功能最好交给 [Gameplay Ability System](GAS.md) 处理）：
```cpp
using namespace UE5Coro::Latent;

// 一个持续半秒的线性冲刺原型
TCoroutine<> AActress::Dash(FVector From, FVector To, FForceLatentCoroutine = {})
{
    bCanDash = false; // 本次冲刺进行期间，禁止重叠的冲刺
    auto Cooldown = Seconds(DashCooldownDuration);
    co_await Timeline(this, 0, 1, 0.5, [&](double Alpha)
    {
        FVector Location = FMath::Lerp(From, To, Alpha);
        SetActorLocation(Location);
    });
    DashComplete.Broadcast();
    co_await Cooldown;
    bCanDash = true;
}
```

> [!TIP]
> 如果一个协程唯一的 `co_await` 就是某个 Timeline，直接 `return` 它通常效率更高。
>
> 这个简化示例没有实现单独的冷却或锁定：
> ```cpp
> using namespace UE5Coro::Latent;
>
> TCoroutine<> AActress::Dash(FVector From, FVector To)
> {
>     TCoroutine Coro = Timeline(this, 0, 1, 0.5, [=, this](double Alpha)
>     {
>         FVector Location = FMath::Lerp(From, To, Alpha);
>         SetActorLocation(Location);
>     });
>     Coro.ContinueWithWeak(this, [](AActress* This)
>     {
>         This->DashComplete.Broadcast();
>     });
>     return Coro;
> }
> ```
