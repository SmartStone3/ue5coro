# 隐式 awaiter

[English](../Implicit.md) | 简体中文

当协程 await 某种本功能所支持的引擎类型时，会创建隐式 awaiter。
协程拿不到 awaiter 本身：它不能被预先创建，也不能存进变量。

通常以可等待对象作为参数的聚合 awaiter（WhenAny、WhenAll 等）可以直接接受这些类型。

<a id="tcoroutinet"></a>
### TCoroutine\<T\>

TCoroutine 不是 awaiter，但可以直接 await。
在异步协程和 latent 协程中 await 它，行为有所不同：

* 异步协程会在被 await 的协程完成时立即恢复，恢复所在的线程就是那个协程完成时所在的线程。
* Latent 协程只能在游戏线程上 await 其他协程。

无论哪种情况，await 一个已经完成的协程都是即时的，并在调用线程上同步继续执行。

await 表达式的结果是 T。
如果被 await 的 TCoroutine 是非 const 右值且 T 可移动，这个值会从协程的返回值存储中移出；
否则它是一份拷贝。
await TCoroutine\<\>（T=void）不产生值，左值和右值 await 的行为完全相同。

示例：
```cpp
using namespace UE5Coro;

TCoroutine Coro1 = []() -> TCoroutine<int>
{
    co_return 10;
}();
TCoroutine Coro2 = [](TCoroutine<int> C) -> TCoroutine<>
{
    int Ten = co_await C;
}(Coro1);
```

### UE::Tasks::TTask\<T\>

await 一个尚未完成的 TTask，会在该任务完成后，把协程的执行转移到 UE::Tasks 系统中
（值得注意的是，这意味着它不再位于游戏线程上）。
作为一项优化，如果 TTask 已经完成，协程会在同一线程上同步继续执行。

await 表达式的结果是 T&，而不是 T，与 TTask\<T\>::GetResult() 保持一致。

示例：
```cpp
using namespace UE::Tasks;
using namespace UE5Coro;

TCoroutine<> Example(TTask<int> Task)
{
    int& Value1 = co_await Task;
    // Task 返回引用，并不意味着你必须用引用接收
    int Value2 = co_await Task;
}
```

### TFuture\<T\>

> [!WARNING]
> TFuture 的 API 在引擎本身中就不稳定，不推荐使用。

await 一个 TFuture 会消耗掉它，并在它完成后恢复协程，
恢复所在的线程就是 TFuture::Then 或 Next 会使用的那个线程。
如果在 co_await 时 future 已经就绪，协程会在当前线程上同步继续执行。

由于这是一个破坏性操作，future **必须**是右值。
必要时请把 future 移动进 await 表达式。
由于引擎的限制，不支持 TSharedFuture\<T\>。

await 表达式的结果是 T，如果可能，它会从 future 中移出，而不是复制。
如果 T 是引用，结果就是指向 TFuture 所引用的同一个对象的引用：不涉及复制或移动。

await TFuture\<void\> 的结果是 void，而不是 TFuture 通常提供的那个毫无意义的 int 值。

示例：
```cpp
using namespace UE5Coro;

TCoroutine<> Example(TPromise<int>& Promise, TFuture<int> Future)
{
    int Value1 = co_await Promise.GetFuture(); // OK，本来就是右值
 // int Value2 = co_await Future; 无法编译
    int Value2 = co_await std::move(Future); // OK，移动进了 co_await
}
```

<a id="delegates"></a>
### 委托

await 一个委托，会在该委托下一次被 Execute() 或 Broadcast() 时恢复协程。
这样做会对委托执行 Bind 或 Add，并在协程恢复时 Unbind/Remove 这个绑定。

> [!NOTE]
> 要 await 一个期望传入已绑定委托的引擎函数，请使用
> [Async::Chain](AsyncChain.md)。

支持以下引擎委托，参数个数不限，有无返回值均可：
* TDelegate（DECLARE_DELEGATE、~~DECLARE_TS_DELEGATE~~[^ts]）
* TMulticastDelegate（DECLARE_MULTICAST_DELEGATE、DECLARE_TS_MULTICAST_DELEGATE）
* 由 DECLARE_DYNAMIC_DELEGATE 创建的类型
* 由 DECLARE_DYNAMIC_MULTICAST_DELEGATE 创建的类型
* 由 DECLARE_DYNAMIC_MULTICAST_SPARSE_DELEGATE 创建的类型
* 由已弃用的 DECLARE_EVENT 创建的类型

[^ts]: Unreal 并没有非多播的 DECLARE_TS_DELEGATE 宏，但它们逻辑上会展开成的
       TDelegate 依然受到支持。

参数必须是 _MoveConstructible_（可移动构造）的。
返回类型必须是 _DefaultConstructible_（可默认构造）的，或者是 void。
支持超过九个参数；以这种方式与 DYNAMIC 委托交互，完全不需要 UFUNCTION，
甚至不需要 UCLASS。

> [!CAUTION]
> 线程安全和同步由你负责：在 await 表达式开始（Bind/Add）或结束
> （Unbind/Remove/UObject 失效）时，不会对数据竞争做任何检查或其他防范措施。
>
> 协程在从委托上解绑之后又被再次恢复，是未定义行为。
> 如果有东西在非 DYNAMIC 委托处于绑定状态时复制了它，然后使用这份副本，
> 就可能发生这种情况。
>
> 类似地，协程取消本身是线程安全的，但大多数 Unreal 委托不是。
> 正在 await 委托的协程，会在触发该委托的线程或取消该协程的线程上解绑，
> 以先发生者为准。

委托会直接、同步地调用进协程。
如果委托被销毁或者从未被调用，协程就不会被恢复，这可能导致内存泄漏。
委托在被 await 期间被销毁是未定义行为。

await 委托支持加速取消。
取消 TCoroutine 可以避免内存泄漏。

await 表达式的结果是一个未指明的类型，可以配合结构化绑定，按需接收委托的参数。
引用参数会以引用形式传递，并且可以写入。
这些写入会通过委托调用传回原先被引用的变量。

如果协程不想接收参数，可以放心地丢弃 await 表达式的值。
不支持以结构化绑定声明之外的任何方式使用 await 表达式的返回值
（包括把它整个存进一个 `auto` 局部变量）。

引用和指针的有效性取决于委托的调用方，但即使是生命周期最短的引用，
也会在下一次 co_await 或 co_return 之前保持有效。

对于返回类型不是 void 的委托，会在协程下一次 co_await 或 co_return 时，
向调用方返回一个默认构造的值。
即使协程 co_return 了另一个结果，委托收到的也是这个默认构造的值。
委托的返回类型与协程的结果类型彼此独立。

示例：
```cpp
using namespace UE5Coro;

class FExample
{
    TDelegate<FString(FName, bool, int&)> Delegate;

    TCoroutine<int> Foo()
    {
        UE_LOGFMT(LogTemp, Display, "First");
        auto&& [Name, bSomething, OutValue] = co_await Delegate;
        OutValue = 1;
        UE_LOGFMT(LogTemp, Display, "Third");
        co_return 0;
    }

public:
    void Bar()
    {
        Foo();
        UE_LOGFMT(LogTemp, Display, "Second");
        int Value = 0;
        FString EmptyString = Delegate.Execute("Name", true, Value);
        // Value == 1
        UE_LOGFMT(LogTemp, Display, "Fourth");
    }
};
```

<a id="generic-workarounds"></a>
#### 通用变通方案

对于 [Async::Chain](AsyncChain.md) 和上面介绍的委托 co_await 功能都不支持的
基于回调的函数，或者对它们是否合适、是否安全存疑时，
可以把 [FAwaitableEvent](Threading.md#fawaitableevent) 用作一种通用的、
线程安全的变通方案的一部分。

如果回调被调用时它正在被 await，`Trigger()` 会同步地回调进协程；
否则，如果回调已经发生过，它会让协程同步地通过。

下面这种技巧应该适用于几乎所有支持 lambda 的场合：

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    FAwaitableEvent Event;

    int Data1, Data2;
    imaginary_library_delegate_t<int, int> Delegate([&](int Param1, int Param2)
    {
        Data1 = Param1; // FAwaitableEvent 本身是线程安全的，
        Data2 = Param2; // 但这两个仍然需要你自己负责！
        Event.Trigger();
    });
    some_imaginary_library_function(Delegate);
    // 直接传入 lambda 的情况大致相同：
    // another_imaginary_library_function([&](int Param1, int Param2){...});
    co_await Event;
    // 在这里 Data1 和 Data2 一定是有效的
}
```

下面这个复杂示例演示了如何对一个需要被继承的类接口类型使用 `shared_ptr`，
以及 `co_await` 是可选的：

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    struct FMyListener final : imaginary_library_listener_t
    {
        std::shared_ptr<FAwaitableEvent> Event;
        virtual void execute_callback() override { Event->Trigger(); }
    };

    // FAwaitableEvent 不可移动、不可复制，但 shared_ptr 可以
    std::shared_ptr<FAwaitableEvent> Event = std::make_shared<FAwaitableEvent>();
    FMyListener Listener;
    Listener.Event = Event;
    some_imaginary_library_function(std::move(Listener));

    if (bSomething)
        co_await *Event;

    // 即使 !bSomething，shared_ptr 也会让事件在这里保持存活，但取决于
    // some_imaginary_library_function 的语义，本协程结束时 Listener
    // 离开作用域这件事可能需要显式处理。
}
```

很多 C 库接受的是裸函数指针，无法使用 lambda 捕获，
但它们通常支持指定一个自定义的 `void*`，并将其传回给函数：

```c++
using namespace UE5Coro;

TCoroutine<> Example()
{
    FAwaitableEvent Event;
    void (*Fn)(void*) = [](void* UserData)
    {
        static_cast<FAwaitableEvent*>(UserData)->Trigger();
    };

    some_imaginary_c_library_function(Fn, &Event);
    co_await Event; // 无条件的 co_await 让 &Event 在这里保持有效
    // 如有需要，在这里从 C 库中注销该函数指针
}
```
