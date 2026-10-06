# 协程

[English](../Coroutine.md) | 简体中文

虽然生成器也是协程，而且 UE5Coro 提供的协程可以与任何其他库和/或你自己的实现混用；
但为清晰起见，本文档中的“协程”一词，专指函数体内含有 co_await 表达式或
co_return 语句、并返回 UE5Coro::TCoroutine\<\> 或其他兼容类型的函数。
“子程序”（subroutine）一词则指函数体内不含 co_await、co_yield 或 co_return 的函数，
无论其返回类型是什么。

## TCoroutine

TCoroutine 可以复制，代表一次单独的协程调用。
用相同的参数（或不带参数）多次调用同一个协程，每次都会返回不同的值。
这些对象（与 TGenerator 不同）并不代表对协程的所有权，
如果不需要，可以放心地丢弃它们，不会影响正在运行的协程。

不支持直接操作协程的 std::coroutine_handle，这样做几乎必然导致未定义行为。

允许复制 TCoroutine 值：副本指向同一次调用，并与原值比较相等。
TCoroutine 有一个没有实际意义、但定义良好的严格全序，因此适合作为有序容器的键。
对于无序容器，也支持 GetTypeHash() 和 std::hash。
TCoroutine 值会一直有效，即使对应的协程早已结束。

### FVoidCoroutine

FVoidCoroutine（出于需要，位于全局命名空间）是 TCoroutine\<\> 的 USTRUCT 封装，
用于绕开 UHT 的限制。
只要可能，就应优先使用 TCoroutine，因为它用起来更安全。

由于引擎的限制，FVoidCoroutine（而非 TCoroutine）可以被默认构造，
而不代表任何实际的协程。
试图操作默认构造的 FVoidCoroutine 背后的协程是未定义行为，
但这些值仍然可以相互比较，也可以检查是否有效。
有效的 FVoidCoroutine 可以安全地转换为 TCoroutine\<\> 并不受限制地使用。
由协程（而非子程序）返回的 FVoidCoroutine 总是有效的。

对于协程，FVoidCoroutine 和 FForceLatentCoroutine（见下文）引脚在蓝图节点图中默认隐藏。
如果一个函数在蓝图中被使用之后才改成协程，隐藏的引脚可能会出现在节点图上，
但这是无害的，只要这些引脚保持未连接，就不会影响行为。
如有需要，可以重新创建受影响的节点来修正其外观。

更多文档见[下文](#fvoidcoroutine-1)。

### TManualCoroutine

TManualCoroutine 不能从协程中返回。
它可以默认构造，并会自动启动属于自己的协程，
之后通过在句柄上调用 SetResult、TrySetResult 或 Cancel 来手动控制该协程。

详细文档见[下文](#tmanualcoroutinet)。

## 结果类型

TCoroutine\<T\> 让协程可以 co_return T。
TCoroutine\<\> 与 TCoroutine\<void\> 是同一个类型。
如果编译器能推导出类型，\<\> 可以省略：
`TCoroutine Coro = Foo();`

（非 void 的）协程结果类型至少必须是 _DefaultConstructible_（可默认构造）、
_MoveAssignable_（可移动赋值）且 _Destructible_（可析构）的。
完整功能还要求 _CopyConstructible_（可复制构造）。
协程有可能在没有提供返回值的情况下结束。
这种情况下，结果是一个默认构造的 T。

TCoroutine\<T\> 可以隐式转换为 TCoroutine\<\>，
从而提供一个擦除了结果类型的协程调用句柄。
把 TCoroutine\<T\> 对象切片为 TCoroutine\<\>，
再用 static_cast 转回原来的类型（T 完全相同）是安全的。

不提供在运行时获知 T 的 RTTI 或反射功能。

## 执行模式

TCoroutine 由两套看起来像一套的独立系统驱动：

|                           |异步模式|Latent 模式                      |
|---------------------------|--------|---------------------------------|
|UObject 所有者/`this`      |忽略    |[追踪生命周期](#latent-lifetime) |
|协程生命周期               |独立    |由引擎持有                       |
|主线程（home thread）      |任意    |游戏线程                         |
|蓝图节点                   |普通    |普通或 🕒                         |
|多线程                     |✅       |✅                                |
|启动协程                   |🐇       |🐢                                |
|await `TLatentAwaiter`     |🐢       |🐇                                |
|await 其他任何东西         |🐇       |🐇                                |

协程使用哪种模式在编译期决定，依据如下逻辑：

1. 如果协程有一个退化（decay）后为 FLatentActionInfo、UE5Coro::TLatentContext
   或 FForceLatentCoroutine 的参数，它就会以 latent 模式运行。
   带有超过 1 个这类参数将无法编译。
2. 如果没有上述任何特殊参数，它会以异步模式运行。
3. 某些函数（例如 UE5CoroGAS 各类上的虚函数）有自己的自定义规则，另行说明。

在返回兼容 TCoroutine 类型的函数中使用 co_yield 将无法编译。
子程序可以接受或返回这些类型中的任何一个，并且不受影响，
直到函数体内出现 co_await 和/或 co_return 而把它们变成协程为止。
由于有返回类型这一要求，只要其他协程实现不去处理 TCoroutine，
UE5Coro 就能与它们在同一个项目中和平共处。

<a id="async-mode"></a>
### 异步模式

这是两者中较简单、性能更好的一种：它与一个独立的 C++ 函数的执行方式非常相似。
函数的生命周期不受管理，它要自己负责清理。
类成员函数还要自己负责追踪所属对象的生命周期：
如果协程会访问 `this`，在已删除的对象上恢复协程，与在其上调用函数一样糟糕。

> [!WARNING]
> 用 C++ 协程手动管理生命周期极其困难。
> 推荐使用 latent 模式的非静态 UCLASS 成员协程，即使是做多线程也一样。

异步模式下的 BlueprintCallable 和/或 BlueprintPure 协程 UFUNCTION，
会在第一次 co_await 或 co_return 时同步地继续执行蓝图，协程的其余部分则独立执行。

异步模式的协程在需要时会自动启动一个 latent 动作
（通常是因为 await 了一个匹配 TLatentAwaiter 概念的类型）。
这比 latent 协程复用已有的 latent 动作开销更大，而且需要访问 world，
world 从 GWorld 读取。

因此，只有在 GWorld 有效时，异步协程才能 await TLatentAwaiter。
通常，如果协程在尝试与 latent 动作交互之前先转移到了游戏线程
（这本来就是那些 awaiter 的要求），GWorld 就是有效的，
但也存在一些 GWorld 无效的特殊情况。
这种情况下，请尝试把 co_await 挪到另一个时间点；如果有异步等价物，
就改用受影响 latent awaiter 的异步等价物；或者把协程的执行模式改为 latent。
不这样做可能导致未定义行为。

当控制流离开协程体时，与协程关联的大部分内存都会被释放：
要么是显式地通过 co_return，要么是对于 TCoroutine\<\>，隐式地运行到最后的 `}`。
这发生在协程最后运行所在的线程上。
只要还有至少一个 TCoroutine 指向这次协程执行，就会保留一小块内存。
例如，co_return 的结果就存放在这里。

<a id="latent-mode"></a>
### Latent 模式

这种模式尽可能贴近蓝图的行为，而非 C++ 的行为。
它注重便利性，会自动处理与背后 latent 动作相关的各种工作，并追踪其目标 UObject 的生命周期。
这种追踪的好处在[下一节](#latent-lifetime)讨论。

函数会查找为所给 FLatentActionInfo 注册的 latent 动作，如果已经存在，
它会**什么都不做直接返回**，这与引擎提供的大多数 latent UFUNCTION 的行为一致。
否则，它会注册一个 latent 动作，协程的生命周期由 latent 动作管理器控制。
如果没有提供 FLatentActionInfo 参数，就不会进行这种防重复处理。

FPendingLatentAction 的几个虚函数以 [latent 回调](LatentCallback.md)的形式
暴露给了协程，但很少需要用到这项功能。
在协程体内借助 `ON_SCOPE_EXIT` 实现 RAII 就能完成大部分工作。

如果提供了 FLatentActionInfo 或 TLatentContext 参数，
它决定了协程在引擎 latent 动作管理器中的目标，以及它的 world context。
否则，对于 FForceLatentCoroutine，world 和目标由函数的第一个参数决定
（对于非静态成员，就是 `this`）。
找不到有效的 world 是致命错误。

协程完成时，会从节点的 latent 执行引脚恢复调用它的蓝图。
被取消的协程不会恢复蓝图。
C++ 调用方享受不到这一点（除非使用 Latent::Chain），
但与蓝图不同，它们可以操作函数的返回值。

> [!NOTE]
> 虽然预期接受 FLatentActionInfo 的协程会被用来为蓝图实现 latent UFUNCTION，
> 但这并不是必需的。
>
> 面对一个没有必要 meta 说明符的 UFUNCTION，UE5Coro 也会照常工作，
> 但这会产生一个不带 🕒 的蓝图节点，调用时不会为 latent info 参数提供有效的值。
> 这不太可能正确工作。
>
> 对于从 C++ 调用的 latent 协程，推荐使用 TLatentContext 和 FForceLatentCoroutine。

Latent 协程通常预期会与游戏线程交互，并且由 latent 动作管理器持有。
对于匹配 UE5Coro::TLatentAwaiter 概念的 awaiter，有一条快速路径，
可以避免为每个 awaiter 创建一个 latent 动作（这与内置的 latent 蓝图节点不同，
后者通常每次调用都会创建一个新的）。
TLatentAwaiter 还会在一个 tick 内对[取消](Cancellation.md)做出反应，
而不是在不确定的一段时间之后。

await 其他任何东西都会让协程进入一种特殊模式，
在这种模式下，它的生命周期被暂时延长到超出其背后 latent 动作的生命周期。
如果引擎在此状态下决定销毁 latent 动作，可以保证协程保持有效足够长的时间，
让清理工作（销毁局部变量等）在协程本身被销毁之前安全地运行。
发生这种情况时，游戏线程**不会**被阻塞。
协程回到游戏线程后，会恢复其正常执行，并把它的生命周期交还给 latent 动作管理器。

latent 动作管理器因任何原因删除协程的 latent 动作，都算作一次强制取消，
会忽略任何取消防护，以确保协程完成清理。
无论 latent 协程原本运行在哪个线程上，**由取消引起的**清理总是在游戏线程上进行。

> [!NOTE]
> 如果一个 latent 协程在不处于游戏线程时运行到结束，清理会在那个线程上进行，
> 然后协程才被视为完成，而不是在游戏线程上清理。
> 这是 C++ 语言规则决定的，无法改变。
>
> 如果不希望这样（例如作用域内还有 latent awaiter），
> 请务必在协程结束前使用 `co_await MoveToGameThread()`。

<a id="latent-lifetime"></a>
#### Latent 生命周期

由 latent 动作管理器控制 latent 协程的生命周期大有裨益，
因为它提供了一定程度的保护，防止在无效对象上运行，进而导致数据损坏和/或崩溃。

最好的说明方式或许是逐行直接对比，展示 latent 协程_不_需要写的那些代码：

<table><tr><td>

```cpp
TCoroutine<> AActress::MemberAsync()
{
    check(IsInGameThread());
    TWeakObjectPtr WeakThis = this;
    co_await MoveToTask();
    auto Value = HeavyProcessing();
    co_await MoveToGameThread();
    if (!WeakThis.IsValid())
        co_return;
    /*this->*/SetValue(std::move(Value));
}
```
</td><td>

```cpp
TCoroutine<> AActress::MemberLatent(FForceLatentCoroutine = {})
{


    co_await MoveToTask();
    auto Value = HeavyProcessing();
    co_await MoveToGameThread();


    /*this->*/SetValue(std::move(Value));
}
```
</td></tr><tr></tr><tr><td> <!-- 额外的一行，用于绕开默认 CSS -->

```cpp
static TCoroutine<> Async(AActress* Target)

{
    check(IsInGameThread());
    TWeakObjectPtr WeakTarget = Target;
    co_await MoveToTask();
    auto Value = HeavyProcessing();
    co_await MoveToGameThread();
    if (!WeakTarget.IsValid())
        co_return;
    Target->SetValue(std::move(Value));
}
```
</td><td>

```cpp
static TCoroutine<> Latent(AActress* Target,
                           FForceLatentCoroutine = {})
{


    co_await MoveToTask();
    auto Value = HeavyProcessing();
    co_await MoveToGameThread();


    Target->SetValue(std::move(Value));
}
```
</td></tr></table>

如果 UObject `this` 或 Target 在协程回到游戏线程之前被销毁，
latent 协程的 latent 动作就会被销毁，进而取消协程的执行，确保 SetValue 不会运行。

由于 UObject 的生命周期和 latent 动作都与游戏线程紧密绑定，
只有在游戏线程上访问协程的目标 UObject 才是安全的。
协程在其他线程上运行时，依然没有什么能阻止垃圾回收的发生。

目前只有协程的目标 UObject（由 FLatentActionInfo 或 TLatentContext 提供，
使用 FForceLatentCoroutine 时则是第一个参数）会以这种方式被追踪。
其他 UObject 可能需要手动使用弱/强指针。

当然，目标 UObject 自身的 UPROPERTY 是由 GC 追踪的，
这让这项功能在实践中对大多数成员协程来说非常方便和强大。
如果一个 latent 协程只与它的目标对象、和/或只通过目标对象的 UPROPERTY
访问的其他对象交互，就不需要任何额外处理。

> [!TIP]
> 协程的局部变量不是 UPROPERTY，所以每次 co_await 之后，
> 总有可能某个未被追踪的 UObject 已经不在了，指针变成了悬空指针。
> `IsValid(Ptr)` 不能用来检查悬空裸指针的有效性。

# TCoroutine

TCoroutine\<\> 是 TCoroutine\<void\> 的便捷写法。
它们是同一个类型。
对于有返回值的协程，可以提供另一个类型参数来代替 void。
这个类型参数至少必须是 _DefaultConstructible_、_MoveAssignable_ 且 _Destructible_ 的。
完整功能还要求 _CopyConstructible_。

控制流在没有 co_return 语句的情况下离开返回 TCoroutine\<T\>（T≠void）的协程体，
是未定义行为，除非协程已被成功[取消](Cancellation.md)。
这种情况下，它的结果是一个默认构造的 T()。

TCoroutine 上有各种方法，可以从不认识协程的同步/阻塞代码中与协程交互。
它也可以被其他协程[await](Implicit.md#tcoroutinet)。

不支持从 TCoroutine 访问 std::coroutine_handle 或协程的 promise，
这样做很可能导致未定义行为。

## 同步 → 异步的转换

除了调用协程这种显而易见的方式之外，还有几个便捷写法，
可以在不想编写完整协程时生成一个 TCoroutine。

### static const TCoroutine\<\> TCoroutine\<\>::CompletedCoroutine

这个字段包含一个结果为 `void`、已经成功完成的协程的句柄。
适合用作 TCoroutine\<\> 字段的默认值。

### static const TCoroutine\<\> TCoroutine\<\>::FailedCoroutine

这个字段包含一个结果为 `void`、已经失败的协程的句柄。
适合用作 TCoroutine\<\> 字段的默认值。

### static TCoroutine\<T\> TCoroutine\<T\>::FromResult(T Value)

本函数可以把一个值转换为协程句柄，其表现就好像有一个协程立即成功地
co_return 了这个值。

该协程会是 `IsDone()`、`WasSuccessful()` 的，其结果会从 `Value` 移入。

### static TCoroutine\<std::decay_t\<V\>\> TCoroutine\<\>::FromResult\<V\>(V&& Value)

这个重载可以把一个值转换为协程句柄，其表现就好像有一个协程立即成功地
co_return 了这个值，而且不必在代码中显式指定类型
（用 `TCoroutine<>::` 代替 `TCoroutine<T>::`）。

由于函数参数会退化（decay），即使 V 是左值引用之类，这个方法也能工作。

该协程会是 `IsDone()`、`WasSuccessful()` 的，其结果会从 `Value` 转发而来。

### static TCoroutine\<\>::FromFailure\<T\>()
### static TCoroutine\<T\>::FromFailure()

这两个函数返回一个协程句柄，其表现就好像有一个协程立即失败、
没能 co_return 一个 T 类型的值。
它们的行为完全相同。

这样的协程会是 `IsDone()` 的，但**不是** `WasSuccessful()` 的，
其结果是一个默认构造的 `T()`。

`FromFailure<void>()` 返回的协程与
[FailedCoroutine](#static-const-tcoroutine-tcoroutinefailedcoroutine) 相似，但并不相等。

## 异步 → 同步的转换

在同步代码中使用一个异步操作可能会有问题。
TCoroutine 除了可以被其他协程 await 之外，还提供了常见的阻塞或注册回调的原语。

### const T& TCoroutine\<T\>::GetResult() const

阻塞调用方直到协程完成，并返回协程结果的引用。
由于可能有多个协程句柄在观察同一个协程，这是一个 const 引用。

### T&& TCoroutine\<T\>::MoveResult()

阻塞调用方直到协程完成，并提供一个可以直接移走的、指向其结果的右值引用。
调用方需要确保只有一个线程这样做，并且最多做一次。
调用 MoveResult 之后，不允许再调用 GetResult 或 MoveResult。

### bool TCoroutine\<\>::IsDone() const

如果协程因任何原因（包括被取消）已经结束，返回 `true`。
读取这个标志不会阻塞。

### bool TCoroutine\<\>::WasSuccessful() const noexcept

如果协程已经成功完成，返回 `true`。
成功完成之后的取消是空操作，不会影响这个值。
读取这个标志不会阻塞。

### bool TCoroutine\<\>::Wait(uint32 WaitTime, bool bIgnoreThreadIdleStats)

本函数会阻塞调用方，等待协程完成，最多等待 `WaitTime` 毫秒（默认为无限）。

关于 `bIgnoreThreadIdleStats` 参数，请参见引擎中的 `FEvent::Wait`。

返回 IsDone()，即成功时为 `true`，超时为 `false`。

### void TCoroutine\<?\>::ContinueWith(? Callback)

这个函数在 TCoroutine\<\> 和 TCoroutine\<T\> 上都有多个重载。
它们接受一个可调用对象，该对象要么不接受参数，要么接受一个与 T 兼容的参数，
并在协程完成时调用它。
对已经完成的协程调用 ContinueWith，会立即执行回调。

ContinueWith 是线程安全的，并保证 `Callback` 恰好执行一次
（除非协程永远不结束，那样就是零次）。

如果协程当前正在运行或已挂起，回调会在协程结束所在的线程上调用
（对于 latent 协程，总是游戏线程）；如果协程已经完成，则在当前线程上同步调用。

示例：
```cpp
using namespace UE5Coro;

TCoroutine<int> Coro = ...;
// 接收结果
Coro.ContinueWith([](int Result)
{
    UE_LOGFMT(LogTemp, Display, "Coroutine completed with result {0}", Result);
});
// 忽略结果
Coro.ContinueWith([]
{
    UE_LOGFMT(LogTemp, Display, "Coroutine completed");
});
```

### void TCoroutine\<?\>::ContinueWithWeak(? StrongPtr, ? Callback)

这个函数同样有许多重载。

第一个参数是指向某个对象的强指针，例如 UObject\*、TSharedPtr 或 std::shared_ptr。
回调可以接受 0、1 或 2 个参数：无参数；一个与 T 兼容的参数；
或者一个与强指针兼容的参数再加一个与 T 兼容的参数。

协程会对该对象持有相应的弱引用（TWeakObjectPtr、TWeakPtr、std::weak_ptr），
并且只有在协程完成时该对象仍然存活，才会执行回调。

与 ContinueWith 一样，本函数是线程安全的，可以在协程完成之前或之后、
在任何线程上随时调用，但它不会给强指针本身赋予额外的线程安全性：
UObject* 必须在游戏线程上使用，NotThreadSafe 的 TSharedPtr
要求协程自身在完成之前做好适当的同步（或者是单线程的），等等。

回调会在协程结束所在的线程上调用（对于 latent 协程，总是游戏线程）；
如果协程已经结束，则在当前线程上同步调用。

示例：
```cpp
using namespace UE5Coro;

void AActress::Subroutine(TSharedPtr<SButton> Button)
{
    TCoroutine<int> Coro = ...;
    Coro.ContinueWithWeak(this, [this]
    {
        UE_LOGFMT(LogTemp, Display, "Coroutine completed");
        check(IsValid(this));
    });

    // 不在 lambda 中捕获 TSharedPtr，按钮就可以在别处被销毁，
    // ContinueWithWeak 只会在 SButton* 有效时运行 lambda
    Coro.ContinueWithWeak(Button, [](SButton* InButton, int Result)
    {
        UE_LOGFMT(LogTemp, Display, "Coroutine completed with result {0}", Result);
        InButton->SimulateClick();
    });
}
```

## 取消

### void TCoroutine\<\>::Cancel()

把协程标记为已取消，这会使它当前的（如果已挂起）或下一次的（如果没有挂起）
co_await 转去清理，而不是继续执行。
协程有阻止或推迟这类取消请求的机制。

取消有[单独的文档页面](Cancellation.md)。

## 其他功能

### FString TCoroutine\<\>::GetDebugName() const

返回协程用 SetDebugName 为自己设置的那个字符串；
在 Shipping 构建中返回空字符串，除非启用了[协程追踪](GameplayDebugger.md#setup)。

与 SetDebugName 之间不是线程安全的。

### static void TCoroutine\<\>::SetDebugName(FString Name)

在返回 TCoroutine 的协程内部调用时，为当前正在执行的协程附加一个调试名，
它会由 UE5Coro.natvis 以及 UE5Coro [gameplay debugger](GameplayDebugger.md) 显示。
默认情况下，本函数在 Shipping 构建中什么也不做，
但如果启用了[协程追踪](GameplayDebugger.md#setup)，它就会生效。

推荐在协程的第一次 co_await 之前调用 SetDebugName。
在协程之外调用 SetDebugName 是未定义行为。

示例：

```c++
// UE5Coro 有意不提供这样的宏，
// 但你可以随意定义自己的！
#define MY_SET_COROUTINE_FUNCTION_NAME() do if constexpr (UE5CORO_DEBUG)       \
    ::UE5Coro::TCoroutine<>::SetDebugName(TEXT("") __FUNCTION__); while (false)
#define MY_FORMAT_COROUTINE_NAME(Fmt, ...) do if constexpr (UE5CORO_DEBUG)     \
    ::UE5Coro::TCoroutine<>::SetDebugName(FString::Format(TEXT(Fmt),           \
                                          {__VA_ARGS__})); while (false)

using namespace UE5Coro;

TCoroutine<> Example()
{
    MY_SET_COROUTINE_FUNCTION_NAME();
    co_await Latent::Seconds(1);
}

TCoroutine<> DynamicExample()
{
    for (int i = 0; i < 100; ++i)
    {
        MY_FORMAT_COROUTINE_NAME("DynamicExample {0}% complete", i);
        co_await Latent::Seconds(0.01);
    }
}
```

### TCoroutine::operator==, TCoroutine::operator<=>, GetTypeHash, std::hash

TCoroutine 适合用作有序容器的键，并且为 Unreal 和 STL 的无序容器提供了哈希。
TCoroutine 之间的顺序没有实际意义，但它在所有类型参数之间都是严格的全序。

# FVoidCoroutine

这个类型是 TCoroutine\<\> 的 USTRUCT 封装，两种类型可以相互隐式转换。
它的主要目的是让人可以编写 UFUNCTION 协程；当它用作这类函数的返回值时，
会对蓝图隐藏。

由于 Unreal 的限制，这个类型可以默认构造，并且不支持 co_return 值。

默认构造的 FVoidCoroutine，或由它转换而来的 TCoroutine 是无效的；
试图使用会访问空的底层协程的功能是未定义行为。
这些对象只应作为协程的返回值或其副本来创建。

要从 latent UFUNCTION 返回一个值，请使用引用参数：
```cpp
UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine ExampleWithValue(int& Value, FLatentActionInfo LatentInfo)
{
    co_await Latent::NextTick();
    Value = 1;
}
```

![上面 ExampleWithValue 函数对应的 latent 蓝图节点](../latent_node_with_value.png)

## bool FVoidCoroutine::IsValid() const

如果该对象背后有一个协程，返回 true；如果它是默认构造的，返回 false。
如果对象有效，就可以安全地将它转换或对象切片为 TCoroutine\<\>。

# FForceLatentCoroutine

这个 USTRUCT 是空的，但只要它出现在协程的参数列表中，协程就会以 latent 执行模式运行。
这样既能享受 latent 动作管理器提供的对象生命周期保护，
又能提供一个可以从 C++ 轻松调用的函数。
接受这个参数的 BlueprintCallable 函数**不会**生成 latent 节点，
尽管它是作为 latent 动作运行的。
latent 动作的目标是函数的第一个参数（对于非静态成员是 `this`），
latent 动作会在该参数的 GetWorld() 返回的 world 中运行。

推荐给它一个默认值，这样使用时它就是隐藏的：
```cpp
// 声明
static TCoroutine<> Example(UObject* WorldContext, int A, int B, int C,
                            FForceLatentCoroutine = {});
// 使用
Example(this, 1, 2, 3);
```

# TLatentContext\<T\>

与 FForceLatentCoroutine 类似，这个参数的存在会让协程以 latent 模式运行。
world context 和 latent 动作目标通过这个结构体显式传入，从而禁用自动检测逻辑。

这个结构体可以从 T\* 隐式转换，并为方便起见提供了 `->` 和前缀 `*` 运算符，
让它表现得就像转换前的那个 T\*。
它的主要预期用途是 latent lambda 协程，因为 lambda 的第一个参数永远是
lambda 对象本身，它不适合作为 world context。

> [!WARNING]
> 协程 lambda 非常容易写错，尤其是带捕获的。
> 这种情况下，请确保把它们存放在生命周期足够长的变量中。
> 最佳实践请参考
> [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cpcoro-coroutines)。

例如，下面这个 lambda 会安全地以 latent 模式运行，
实际上让 BeginPlay 可以突破引擎的限制，以协程的方式编写：
```cpp
void AActress::BeginPlay() // 它是重写函数，所以不能返回 TCoroutine<>
{
    [](TLatentContext<AActress> This) -> TCoroutine<>
    {
        co_await Async::MoveToTask();
        auto Data = ComputeExpensiveThing();
        co_await Async::MoveToGameThread();
        This->SetData(Data); // Latent 模式：This 一定有效
    }(this);
}
```

要写一个不关心上下文内容的 latent lambda，可以用下面这种更简短的语法：
```cpp
[](TLatentContext<>) -> TCoroutine<>
{
    co_return; // 协程代码写在这里
}(this);
```

在高级用法中（主要是协程目标没有 world 的情况，例如引擎子系统或 CDO），
可以分别提供目标和 world。
这很危险，需要在生命周期管理上格外小心。
```cpp
[](TLatentContext<>, int Something) -> TCoroutine<>
{
    co_return; // 协程代码写在这里
}({this, MyWorld}, 1);
```

TLatentContext 有意不把它的内容存放在 ObjectPtr 或 UPROPERTY 中。
它唯一的预期用途是直接传给一个 latent 协程，而 latent 协程已经通过引擎的
latent 动作管理器，为协程目标和 world 提供了 UObject 生命周期管理。

# TManualCoroutine\<T\>

另请参见：[FAwaitableEvent](Threading.md#fawaitableevent)，
它是一个为协程提供类似功能的多线程原语。
TManualCoroutine 是 FAwaitableEvent 的加强版封装。

TManualCoroutine\<T\> 只能从子程序中返回（如果要 `co_return` 一个，
返回类型就得是 `TCoroutine<TManualCoroutine<T>>`，这通常没什么用）。
它会立即在当前线程上启动一个协程，该协程会用传入的名字调用
[SetDebugName](#static-void-tcoroutinesetdebugnamefstring-name)，
然后挂起，什么也不做，直到它被取消，或者从外部提供了结果，
其表现就好像它是这样实现的：

```c++
TManualCoroutine<T> IllustrationOnly_WillNotCompile()
{
    TCoroutine<>::SetDebugName(Debug_name_passed_to_constructor);
    co_await SetResult_gets_called_externally; // 取消可能发生在这里
    co_return The_argument_passed_to_SetResult;
}
```

在 [gameplay debugger](GameplayDebugger.md) 中，它们会显示为 `Manual` 协程。

TManualCoroutine\<T\> 可以复制，并且可以安全地对象切片为 TCoroutine\<T\>
或 TCoroutine\<\> 来观察它（切片后的对象同样可以复制）。
但 TCoroutine 不能向下转换回 TManualCoroutine。
与 TCoroutine 一样，所有副本都指向同一次协程执行，
并且这些句柄可以在其他（返回 TCoroutine 的）协程中直接 await。

最后一个 TManualCoroutine 离开作用域或以其他方式被销毁时，会取消协程，
以免它永远滞留而造成内存泄漏。
如果协程已经成功，这次取消就是空操作。

示例：
```c++
// 用子程序封装一个旧式的、基于回调的 API：
TCoroutine<int> ExampleSubroutine()
{
    TManualCoroutine<int> Coro(TEXT("Example name"));
    ExternalDelegate.BindLambda([=](bool bSuccess, int Result)
    {
        if (bSuccess)
            Coro.SetResult(Result);
        else
            Coro.Cancel();
    });
    return Coro; // 有意为之且安全的对象切片
}
```

### TManualCoroutine\<T\>::TManualCoroutine(FString DebugName = \{\})

启动一个手动协程，可选地附带调试名。

### TManualCoroutine\<T\>::~TManualCoroutine()

如果这是指向同一协程的最后一个 TManualCoroutine 对象
（TCoroutine\<T\> 句柄只是观察者，**不算在内！**），则取消该协程。

### void TManualCoroutine\<T\>::SetResult(T Result)

如果协程还没有因为另一次 (Try)SetResult 或 Cancel 调用而完成，
则让协程成功地 `co_return` 所提供的值。

如果协程已经完成，它的状态和结果都不会改变，并且这次调用会触发 `ensure`。

### bool TManualCoroutine\<T\>::TrySetResult(T Result)

与 SetResult 类似，但不会触发 `ensure`。
如果这次调用成功地完成了协程，返回 `true`；
如果协程已经完成（成功或被取消），返回 `false`。
