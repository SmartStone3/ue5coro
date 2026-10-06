# 协程取消

[English](../Cancellation.md) | 简体中文

> [!NOTE]
> 本页的内容都不适用于生成器。
> TGenerator 拥有协程的执行权，销毁它就能达到同样的目的。

返回 TCoroutine 的协程自带集成的取消支持。
被取消的协程，其 `co_await` 会像抛出了一个无法捕获的异常那样改变执行流程[^noexcept]：
局部变量的析构函数会运行，与之关联的内存会被释放，等等。

[^noexcept]: 这里并不涉及异常，即使关闭了异常，这项功能也完全可用。

取消被完全处理之后，TCoroutine::IsDone() 会返回 true，
TCoroutine::WasSuccessful() 会返回 false。
TCoroutine\<T\> 的结果会被设为默认构造的 T()（如果 T≠`void`）。

取消是线程安全的。
异步协程会在取消它的那个线程上清理，或者在它本该继续运行的那个线程上清理。
Latent 协程**被取消时**总是在游戏线程上清理。

> [!NOTE]
> 如果一个 latent 协程在不处于游戏线程时运行到结束，清理会在那个线程上进行，
> 然后协程才被视为完成，而不是在游戏线程上清理。
> 这是 C++ 语言规则决定的，无法改变。
>
> 如果不希望这样（例如作用域内还有 latent awaiter），
> 请务必在协程结束前使用 `co_await MoveToGameThread()`。

如果协程在运行过程中被取消，在它 co_await 某个东西或 co_return 之前，什么都不会发生。
co_return（显式写出的，或者在 T=`void` 时运行到最后的 `}` 而隐式产生的）
会成功完成，并忽略这次取消。

一个正在 await 的协程被取消（无论是在 await 之前还是 await 期间），
会在取消发出之后、到协程正常**恢复**执行之前的某个未指明的时刻处理这次取消。
取决于正在 await 的对象，这可能需要相当长的时间；
如果 awaiter 永远不完成，甚至可能是无限长。

如果是 latent 协程正在 await 一个满足 TLatentAwaiter 概念的对象，
无论 awaiter 是否完成，取消都会在一个 tick 内被处理。

满足 TCancelableAwaiter 概念的 awaiter 同样会很快地处理到来的取消
（不以 tick 计），即使它们本来永远不会恢复协程。

## 由协程发起

`co_await UE5Coro::FSelfCancellation()` 会自我取消，并转入清理，而不是恢复协程。

> [!CAUTION]
> 协程在清理期间（例如在 ON_SCOPE_EXIT 中）取消自己会导致死锁。

### 异步协程

自我取消是即时的，并且在执行 await 的线程上同步进行。

### Latent 协程

如果自我取消发生在游戏线程上，取消会被立即同步处理；
如果发生在其他线程上，游戏线程会在 latent 动作管理器的下一次 tick 时处理它。

如果协程实现的是一个 latent UFUNCTION，它在蓝图中的 latent 输出执行引脚**不会**被触发。
执行会停在调用该协程的那个节点。

如果 latent 协程是 `UFUNCTION(BlueprintCallable)`，但不是 `meta = (Latent)`
（这是受支持的组合），取消在蓝图中没有任何影响：
无论是否被取消，执行引脚都会在第一次 co_await 或 co_return 时同步触发。

## 由外部请求

TCoroutine::Cancel() 请求底层协程停止运行，
如第一节所述，这个请求会在当前（如果协程已挂起）或下一次（如果没有挂起）
co_await 期间的某个未指明时刻得到处理。

取消一个已经完成或即将完成的协程是安全的、线程安全的，而且没有任何效果。
多次取消与一次取消效果相同。

> [!CAUTION]
> 协程在清理期间（例如在 ON_SCOPE_EXIT 中）取消自己会导致死锁。

没有撤回取消的功能。

## 由引擎发起

Latent 协程由其 UWorld 的 latent 动作管理器持有，
管理器可能会在协程运行期间决定 `delete` 掉它们的 latent 动作。
这会被转换为对协程的强制取消。

如果这发生在一个异步协程正在 await TLatentAwaiter 的时候
（这会在幕后创建一个可能被 `delete` 的临时 latent 动作），
那么它引起的是一次普通取消，可以加以防护。
关于取消防护，见下文。

# 辅助功能

还有一些附加功能，让协程可以显式地与自身的取消交互：

## FCancellationGuard

在高级用法中，如果有一段代码必须保证 co_await 会恢复协程，
可以用 `UE5Coro::FCancellationGuard` 推迟到来的取消请求。
使用它之前，请考虑：如果协程的 `this` 被销毁了，协程是否依然有效。

FCancellationGuard 对象只能作为返回 TCoroutine 的协程中的局部变量；
在其他任何地方使用都是未定义行为。

* 只要协程体内有一个或多个这样的对象存活，TCoroutine::Cancel() 请求就会被推迟，
  直到最后一个对象离开作用域；即使被取消，co_await 也会恢复协程。
* 存在有效的取消防护时，尝试自我取消是非法的。
* 如果 latent 协程背后的 latent 动作被引擎 `delete`，它会忽略取消防护。

示例：
```cpp
using namespace UE5Coro;

TCoroutine<FThing> Example()
{
    {
        FCancellationGuard Guard;
        co_await ImportantFunction1();
        co_await ImportantFunction2();
    } // 普通取消只能发生在下一行：
    co_return co_await RegularThing();
}
```

## FOnCoroutineCanceled

这个作用域守卫在协程中的行为与 `ON_SCOPE_EXIT` 类似，
但只有当协程正因取消（强制的 latent 取消或普通取消）而被销毁时，才会运行传入的回调。
大多数情况下，做无条件清理应优先使用 `ON_SCOPE_EXIT`。

如果该对象不是某个返回 TCoroutine（或兼容类型）的协程的局部变量，
回调是否运行是未定义的。

示例：
```cpp
using namespace UE5Coro;

TCoroutine<> Example()
{
    ON_SCOPE_EXIT { UE_LOGFMT(LogTemp, Display, "Finally"); };
    FOnCoroutineCanceled _([] { UE_LOGFMT(LogTemp, Display, "Canceled"); });
    co_await RegularThing();
    UE_LOGFMT(LogTemp, Display, "Successful");
}
```

## 手动检查取消

`UE5Coro` 命名空间中有几个函数可以直接与取消交互。

### auto FinishNowIfCanceled() noexcept

如果你在一个没有自然 co_await 的紧凑循环中运行，却又想处理到来的取消请求，
可以用 `co_await UE5Coro::FinishNowIfCanceled()` 手动处理它们。

* 如果协程没有被取消，它会**同步**、立即继续运行，因此这项检查的开销相对较低。
* 如果协程被取消了，co_await 会照常转入清理。

`FinishNowIfCanceled()` 的返回值可以复制、可以重复使用，但没有实际意义。
无论 await 的是哪个对象，行为都一样：它不能用来观察另一个协程的取消。

取消会被正常处理：FCancellationGuard 会被遵守（到来的 `delete` 等情况除外）。
存在有效的 FCancellationGuard 时也允许使用本函数，此时只有强制取消会被放行。

异步协程会在当前线程上处理取消。
Latent 协程的取消总是在游戏线程上处理。

示例：
```cpp
using namespace UE5Coro;

for (int i = 0; i < Items.Num(); ++i)
{
    ProcessItem(Items[i]);
    // 每处理 128 个元素处理一次取消
    if (i % 128 == 0)
        co_await FinishNowIfCanceled();
}
```

### bool IsCurrentCoroutineCanceled()

本函数只是返回一个 bool，不会处理取消。
本函数能“看穿” FCancellationGuard：如果有到来的取消请求正被推迟，它会返回 `true`。

在兼容 TCoroutine 的协程之外调用它会引发未定义行为。

请优先使用 `co_await FinishNowIfCanceled();`，而不是
`if (IsCurrentCoroutineCanceled()) co_return;`。
