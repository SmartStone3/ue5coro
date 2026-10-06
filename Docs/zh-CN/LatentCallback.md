# Latent 回调

[English](../LatentCallback.md) | 简体中文

`UE5Coro::Latent` 命名空间中的这些类型，用途与引擎的 `ON_SCOPE_EXIT`
以及 `UE5Coro::FOnCoroutineCanceled` 类似，但它们专门针对 latent 协程背后的 latent 动作。
它们不是 awaiter，因此 TLatentAwaiter 对它们不适用。

它们只有作为 **latent** 协程中的局部变量才有意义。
其他任何用法都是未定义行为。
实际上，在无效用法下它们多半什么也不做，但这一点没有保证。

有了协程之后，需要用到这些类型的场合，甚至比手动重写 FPendingLatentAction
中相应函数的场合还要少。
大多数情况下，请优先使用 RAII 和/或 `ON_SCOPE_EXIT` 做无条件清理。

## FOnAbnormalExit

这个类型相当于另外两个类型 FOnActionAborted 和 FOnObjectDestroyed 的组合。

只有当协程因为引擎的 NotifyActionAborted 或 NotifyObjectDestroyed 回调而被销毁时，
传入的回调才会被调用。

无论协程在被取消之前运行在哪里，回调都会在游戏线程上运行。

示例：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine Example(FLatentActionInfo LatentInfo)
{
    ON_SCOPE_EXIT { UE_LOGFMT(LogTemp, Display, "Finally"); };
    FOnCoroutineCanceled Cancel([] { UE_LOGFMT(LogTemp, Display,
                                               "Any cancellation"); });
    FOnAbnormalExit Exit([] { UE_LOGFMT(LogTemp, Display,
        "Only called if the latent action manager initiates cancellation"); });
    co_await Seconds(1);
    UE_LOGFMT(LogTemp, Display, "Success");
}
```

## FOnActionAborted

用法和线程行为与 FOnAbnormalExit 完全相同，但只有当协程因为 latent 动作管理器
中止了它的 latent 动作而被销毁时，回调才会运行。

它暴露的是引擎中的 `FPendingLatentAction::NotifyActionAborted`，可参见该函数。

示例：
```cpp
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine Example(FLatentActionInfo LatentInfo)
{
    FOnActionAborted _([] { UE_LOGFMT(LogTemp, Display, "Action aborted"); });
    co_await Seconds(1);
    UE_LOGFMT(LogTemp, Display, "Success");
}
```

## FOnObjectDestroyed

用法和线程行为与 FOnAbnormalExit 完全相同，但只有当协程因为其目标 UObject
被垃圾回收而被销毁时，回调才会运行。

它暴露的是引擎中的 `FPendingLatentAction::NotifyObjectDestroyed`，可参见该函数。

示例：
```cpp
using namespace UE5Coro::Latent;

UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine Example(FLatentActionInfo LatentInfo)
{
    FOnObjectDestroyed _([] { UE_LOGFMT(LogTemp, Display, "Object destroyed"); });
    co_await Seconds(1);
    UE_LOGFMT(LogTemp, Display, "Success");
}
```
