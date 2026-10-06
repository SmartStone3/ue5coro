# UE5CoroGAS

[English](../GAS.md) | 简体中文

UE5CoroGAS 是一个独立的可选模块，提供围绕引擎 Gameplay Ability 系统的额外功能。
无论出于什么原因不使用 GAS 的项目，都不会从中受益。

要使用它，请启用引擎的 GameplayAbilities 插件，在 Build.cs 中引用 "UE5CoroGAS"，
并使用 `#include "UE5CoroGAS.h"`。

本模块的协程使用特殊的返回类型 `UE5Coro::GAS::FAbilityCoroutine`。
返回该类型的协程运行在一种特殊的
[latent 模式](Coroutine.md#latent-mode)下，会根据所在的类来处理与能力相关的事件。

该类型**唯一**的用途，是作为 UE5CoroGAS 提供的纯虚函数及其重写的返回类型。
在其他任何协程中返回它都是未定义行为。

虽然它可以隐式转换为 TCoroutine\<\>（也可以以该类型到处传递），
但拿到它的返回值并不直接。
而且也没有必要。
与这些协程的交互主要通过它们的 GAS 基类，以及本模块提供的额外集成来完成。

## UUE5CoroGameplayAbility

这个类提供了一个方便的基类，用 C++ 协程来实现异步的 gameplay ability。
不要重写 ActivateAbility，而是用协程重写新的 ExecuteAbility 方法。

> [!CAUTION]
> 用普通函数（非协程）重写 ExecuteAbility 是未定义行为。

在 Unreal Engine 5.3 和 5.4 上支持所有实例化策略（instancing policy），
包括在运行时动态改变实例化策略。

从 5.5 开始不再支持 NonInstanced。
可以用 CVar `AbilitySystem.Fix.AllowNonInstancedAbilities` 把它找回来，
但由于引擎存在问题，不推荐这样做。

下列事件会被转换为与 ExecuteAbility 协程的交互：

* 协程完成时调用 EndAbility。
* 自我取消相当于调用 EndAbility(..., true)。
  推荐用自我取消代替调用 CancelAbility，因为后者要到下一次 co_await 才会被处理。
* 到来的 CancelAbility 调用会取消协程，处理完成后会调用 EndAbility(..., true)。
* 到来的 EndAbility 调用（包括来自 CancelAbility 的）会取消协程。
* EndAbility 是否复制由 UUE5CoroGameplayAbility 上的一个属性控制，
  可以随时自由修改。
  默认会复制。
* FCancellationGuard **不会**影响 CanBeCanceled（但你可以用自己的逻辑自由重写它）。
  取消会被接收，并且（非强制的取消）会被推迟到最后一个防护离开作用域。
* 关于由 GAS 自身驱动、可能导致强制取消的特殊垃圾回收行为，
  见[下文](#garbage-collection-considerations)。

和往常一样，你需要在协程中调用 CommitAbility。
你可以自由重写其他任何未标记为 `final` 的方法。
为保证正确运行，假定你的重写会调用对应的 Super 版本
（ExecuteAbility 除外，它是 PURE_VIRTUAL）。

UUE5CoroGameplayAbility 被标记为 NotBlueprintable，但你可以在子类中改回来。
你需要自己负责从蓝图中正确地与 ExecuteAbility 交互。
ActivateAbility/ActivateAbilityFromEvent 的 BlueprintImplementableEvent
在子类中不会被调用，这类需求请使用普通的蓝图 gameplay ability。

### auto UUE5CoroGameplayAbility::Task(UObject*, bool bAutoActivate = true)

这个 protected 函数接受一个只带一个 BlueprintAssignable 委托的 UObject*，
并返回一个 awaiter，等待该委托被 Broadcast()。

如果 `bAutoActivate` 为 true（默认值），`UGameplayTask` 和
`UBlueprintAsyncActionBase` 会被自动激活：

```cpp
using namespace UE5Coro;

GAS::FAbilityCoroutine UExampleGameplayAbility::ExecuteAbility(...)
{
    // 这里会自动找到并使用 EventReceived 委托
    co_await Task(UAbilityTask_WaitGameplayEvent::WaitGameplayEvent(...));
}
```

这个封装锁定在游戏线程上，会立即响应取消，但也会丢弃委托的参数。
在 gameplay ability 协程中 await 单委托任务时，首选这种方式。
它同样是世界敏感的，详见[此页](Latent.md)。

如果它不适用，请手动激活任务，然后直接
[co_await 它的委托](Implicit.md#delegates)。
这种方式支持委托返回值，但不会追踪 world。
你需要确保：如果任务在完成之前被销毁，协程会被取消或以其他方式恢复。

## UUE5CoroAbilityTask

这个类让你可以用协程实现一个 ability task。
不要重写 Activate，而是用协程重写 Execute 来执行任务，
并重写 Succeeded/Failed 来广播任务需要的委托。

> [!CAUTION]
> 用普通函数（非协程）重写 Execute 是未定义行为。

由于 UnrealHeaderTool 的限制，无法提供一个开箱即用的通用 ability task。
子类需要提供用于创建任务的静态 UFUNCTION，以及用于完成事件的 UPROPERTY 委托。

Unreal 期望 gameplay task 在 EndTask _之后_ 调用它们的委托，
所以请确保委托是在 Succeeded 或 Failed 中广播的，而不是在 Execute 的末尾。
GAS 自身会在 EndTask 中把任务标记为垃圾，因此 Succeeded 或 Failed 运行时，
`this` 已经无效（但还没有被垃圾回收）。

Execute 会在 latent 模式下运行，并带有以下额外集成：

* 协程完成时会调用 EndTask，以及 Succeeded 或 Failed 之一；
  后两者是 `UUE5CoroAbilityTask` 上的虚函数，默认实现为空操作。
  自我取消会触发 Failed 而不是 Succeeded。
* OnDestroy（例如来自 EndTask）会取消协程。

## UUE5CoroSimpleAbilityTask

这个类是 UUE5CoroAbilityTask 的便捷子类，提供了一对通用委托，
分别在 Succeeded 和 Failed 中广播，适用于不需要额外委托或完成逻辑的协程任务。

这个类的子类除了重写 Execute 之外，只需要为蓝图提供静态 UFUNCTION。

<a id="garbage-collection-considerations"></a>
# 垃圾回收注意事项

GAS 自身有时会把能力和任务标记为垃圾。
这通常是在响应 EndAbility、EndTask 或其他形式的引擎发起的取消（例如 PIE 结束）。
值得注意的是，非实例化的 gameplay ability 不受此影响，因为它们运行在 CDO 上。

如果 GAS 把一个对象标记为垃圾，latent 动作管理器会在下一次 tick 时移除它的 latent 动作，
其中就包括在幕后驱动协程的那一个。
由于此时协程已经无法再运行，它会被强制取消，并忽略 FCancellationGuard。
此时可以观察到 `!IsValid(this)`，例如在协程内部局部变量的析构函数中，
或者在 ON_SCOPE_EXIT 之类的其他守卫中。
`UE5Coro::FOnCoroutineCanceled` 和 `UE5Coro::Latent::FOnObjectDestroyed`
会响应这类强制取消。

如果协程被强制取消时不在游戏线程上，取消的处理会（照常）推迟到下一次 co_await，
但到那时 `this` 有可能已经被 `delete` 了。

为了更符合引擎的预期、与蓝图能力/任务的行为保持一致，
UE5CoroGAS 的这些类在强制取消时，相比“常规” latent 模式有以下额外的行为差异：

* UUE5CoroGameplayAbility 不会对自己调用 EndAbility。
* UUE5CoroAbilityTask 不会调用 EndTask、Succeeded 或 Failed。
* UUE5CoroSimpleAbilityTask 不会 Broadcast 任何委托。
