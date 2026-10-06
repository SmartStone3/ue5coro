# 异步 awaiter

[English](../Async.md) | 简体中文

`UE5Coro::Async` 命名空间中的这些 awaiter 主要处理多线程。

下列函数的返回值一般都可以复制、可以重复使用。
多次 await 它们会再次执行同样的动作。

### auto MoveToThread(ENamedThreads::Type) noexcept

await 本函数的返回值，会把协程的执行转移到指定的命名线程上。

如果协程已经在那个线程上，则什么都不会发生：它会继续同步执行。

示例：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Async;

TCoroutine<> ThreadHopperExample()
{
    OnCallerThread();
    co_await MoveToThread(ENamedThreads::RHIThread);
    OnRHIThread();
    co_await MoveToThread(ENamedThreads::AnyBackgroundThreadNormalTask);
    OnBackgroundThread();
    co_await MoveToThread(ENamedThreads::GameThread); // 参见 MoveToGameThread()
    OnGameThread();
}
```

### auto MoveToGameThread() noexcept

这是 `MoveToThread(ENamedThreads::GameThread)` 的便捷写法，用法和行为完全相同。

### auto MoveToSimilarThread()

本函数的返回值会记住它是从哪一类命名线程调用的，之后可以用它回到那个线程，
或与之等价的线程（游戏线程回到游戏线程，渲染线程回到渲染线程，
后台线程回到任意其他后台线程，等等）。

当事先不知道调用线程是哪个时，这很有用。

返回值可以多次 await，每次都会回到最初记录的那个线程。

如果协程已经在那个线程上运行，则什么都不会发生。
因此，`co_await MoveToSimilarThread();` 毫无意义。
应该把返回值保存下来，之后在另一个线程上使用。

例如：
```cpp
using namespace UE5Coro::Async;

auto GoBack = MoveToSimilarThread(); // 在这里记录当前线程
co_await MoveToThread(ENamedThreads::AnyBackgroundThreadNormalTask);
DoBackgroundProcessing();
co_await GoBack; // 回到记录的线程
```

### auto MoveToTask(const TCHAR* DebugName = nullptr)

await 本函数的返回值，会把调用它的协程转移到 UE::Tasks 系统的一个任务中。
传入的调试名会传给该任务。

多次 await 会各启动一个任务，调试名相同。

示例：
```cpp
using namespace UE5Coro::Async;

co_await MoveToTask();
auto Value = DoHeavyProcessing();
co_await MoveToGameThread();
UseValueOnGameThread(std::move(Value));
```

### auto MoveToThreadPool(FQueuedThreadPool& ThreadPool = *GThreadPool, EQueuedWorkPriority Priority = EQueuedWorkPriority::Normal)

本函数的返回值让协程可以以指定的优先级，把执行转移到指定的线程池中。

返回值可以重复使用，它会记住最初传入的线程池和优先级，
每次 await 都会重新排队到同一个线程池上。

如果 co_await 时线程池已经无效，则行为未定义。

示例：
```cpp
using namespace UE5Coro::Async;

TCoroutine<> ProcessOnThreadPool(AActress* Target, FQueuedThreadPool& ThreadPool,
                                 FForceLatentCoroutine = {})
{
    if (!ensure(IsValid(Target)))
        co_return;
    co_await MoveToThreadPool(ThreadPool);
    auto Value = SomeExpensiveFunction();
    co_await MoveToGameThread();
    Target->SetValue(std::move(Value));
}
```

### auto Yield() noexcept

本函数的返回值会把协程转回它原先所在的同一类命名线程（类似 MoveToSimilarThread），
但它保证会挂起协程（这一点与 MoveTo[...]Thread 系列函数不同）。

返回值如果被重复使用，总是让出（yield）回到与 co_await 那一刻的当前线程相似的线程，
也就是说，它不会记录原始线程。
因此，保存返回值意义不大。
创建它几乎没有开销，它也不包含任何数据。

示例：
```cpp
using namespace UE5Coro::Async;

TCoroutine<> Example()
{
    OnCallerThread();
    auto Yielder = Yield(); // 不推荐这样用，仅作演示
    co_await Yield(); // 协程在这里返回到它的调用方
    OnCallerThread();
    co_await MoveToGameThread();
    OnGameThread();
    co_await Yielder; // Yield 不会记录原始线程
    OnGameThread();
}
```

### auto MoveToNewThread(EThreadPriority Priority = TPri_Normal, uint64 Affinity = FPlatformAffinity::GetNoAffinityMask(), EThreadCreateFlags Flags = EThreadCreateFlags::None) noexcept

await 本函数的返回值，会把协程的执行转移到一个新创建的“完整”线程中。
每次 co_await 都会启动一个新线程。
当协程 co_return 或转移到其他线程时，该线程结束。

对于会拖累引擎线程池的长时间阻塞操作，用它很方便。

参数含义见引擎函数 `FRunnableThread::Create()`。

示例：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Async;

TCoroutine<> LongWait(FEvent* Event)
{
    co_await MoveToNewThread();
    Event->Wait(/*无限等待*/);
    co_await MoveToGameThread();
    DoThingsAfterEventTrigger();
}
```

### auto PlatformSeconds(double Seconds) noexcept
### auto PlatformSecondsAnyThread(double Seconds) noexcept
### auto UntilPlatformTime(double Time) noexcept
### auto UntilPlatformTimeAnyThread(double Time) noexcept

这些函数的行为与 UE5Coro::Latent::RealSeconds 和 UE5Coro::Latent::UntilRealTime
有些相似，但它们与线程无关（free-threaded），不需要 world，也不需要背后的 latent 动作。

因此，它们适合用在引擎早期代码中（此时还没有可供 latent 动作使用的 world），
或者不想、不便处理 world 的场合。

时间来源是引擎函数 `FPlatformTime::Seconds()`。

* PlatformSeconds(AnyThread) 会把协程挂起指定的秒数。
* UntilPlatformTime(AnyThread) 会一直挂起，直到 FPlatformTime::Seconds()
  到达指定的时间点。

这些函数返回的 awaiter 支持加速取消。

AnyThread 版本会在后台线程上恢复协程；名字较短的版本会尽量在原来的命名线程上恢复
（游戏线程回到游戏线程，渲染线程回到渲染线程，等等）。<br>
如果不需要这项功能，AnyThread 版本的效率稍高一些。

返回值可以复制、可以重复使用；如果原定的目标时间已经过去，await 它们什么也不做。
复制返回值会复制原始调用中的目标时间和期望线程。

时间从函数调用那一刻开始计算。示例：

```cpp
using namespace UE5Coro::Async;

auto Awaiter = PlatformSecondsAnyThread(2);
co_await PlatformSeconds(0.3);
co_await Awaiter; // 大约 1.7 秒后恢复
co_await MoveToGameThread();
co_await Awaiter; // 空操作
check(IsInGameThread());
```
