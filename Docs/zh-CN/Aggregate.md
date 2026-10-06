# 聚合 awaiter

[English](../Aggregate.md) | 简体中文

这些 awaiter 让你可以把多个可等待对象（awaitable）或 TCoroutine 组合成一个操作。

它们支持加速取消（expedited cancellation）。

这些函数的返回值可以复制，并允许同时有一个 co_await。
第一次 await 完成后，后续的 await 会在调用线程上同步返回相同的值
（WhenAll 除外，它是 `void`）。

await WhenAny、WhenAll 和 Race 的协程，会在与某个传入参数相对应的线程上恢复，
或者在取消该协程的那个线程上恢复。
例如，如果所有参数被直接 await 时都会在游戏线程上恢复，并且协程从未被取消，
或者只在游戏线程上被取消（常见情况），那么它们的聚合保证在游戏线程上恢复；
但如果其中某个参数会在别的线程上恢复，或者协程可能从别的线程被取消，
那么它会在游戏线程或那个别的线程上恢复。

WhenAny 和 WhenAll 的 Latent 版本总是在游戏线程上恢复。

### auto WhenAny(TAwaitable auto&&... Awaitables)
### auto WhenAny(const TArray\<TCoroutine\<\>\>& Coroutines)

这些函数会 `co_await` 传入的所有对象，并在第一个完成时恢复调用方协程。
其余的会被忽略，但它们仍处于被 co_await 的状态，这对于不能重复使用的类型很重要。

“完成”也包括**不**成功的完成。

await 这些函数返回值的结果，是最先完成的那个参数的索引。
两个重载的行为完全相同。
如果参数个数为零，会在调用线程上立即返回一个负值（0 表示第一个参数）。

简单示例：
```cpp
using namespace UE5Coro;

int FirstTask = co_await WhenAny(TaskA /*0*/, TaskB /*1*/, TaskC /*2*/);
```

复杂示例：
```cpp
using namespace UE5Coro;

auto Awaiter1 = SomeAsyncTask();
auto Awaiter2 = AnotherAsyncTask();
DoSomethingUsefulBeforeAwaiting();
int FirstAwaiter = co_await WhenAny(std::move(Awaiter1), std::move(Awaiter2));
```

### auto Latent::WhenAny(TLatentContext\<const UObject\> LatentContext, TAwaitable auto&&... Awaitables)

仅供在游戏线程上进行高级用法。
本函数的工作方式与普通 WhenAny 相同，但各参数会以 latent 模式、
使用传入的上下文被 await。

示例：
```cpp
using namespace UE5Coro::Latent;

bool bTimedOut = !!co_await WhenAny(this, Task, Latent::Seconds(1));
```

### auto Race(TArray\<TCoroutine\<\>\> Coroutines)
### auto Race(TCoroutine\<T\>... Coroutines)

这些函数的行为与 WhenAny 类似，但第一个完成的协程会取消其他所有协程。
“完成”也包括**不**成功的完成。

参数只能是 TCoroutine，不能是任意可等待对象，因为 awaiter 不能被直接取消。
两个重载的行为完全相同。

如果参与竞速的协程个数为零，Race 会立即成功，co_await 表达式返回一个负值
（0 表示第一个协程）。

没有 RaceLatent。
输入的协程已经各自确定了执行模式。

取消一个正在 await Race() 的协程，会立即处理取消。
如果 Race 的返回值因任何原因被销毁，这场竞速以及参与其中的所有协程都会被取消，
即使它从未被 await。

示例：
```cpp
using namespace UE5Coro;

int Action = co_await Race(TryAttacking(), TryHiding(), TryJumping());
bool bJumped = (Action == 2);
```

### auto WhenAll(TAwaitable auto&&... Awaitables)
### auto WhenAll(const TArray\<TCoroutine\<\>\>& Coroutines)

这些函数会 co_await 传入的所有对象（不可重复使用的 awaiter 会被消耗掉），
并在最后一个完成后恢复调用方协程。

“完成”也包括**不**成功的完成。

如果参数个数为零，操作会立即成功，协程在调用线程上同步继续执行。

示例：
```cpp
using namespace UE5Coro;

TArray<TCoroutine<>> Tasks;
for (int i = 0; i < 100; ++i)
    Tasks.Add(ExpensiveAsyncCoroutine(i));
DoSomethingUsefulBeforeAwaiting();
co_await WhenAll(Tasks);
```

### auto Latent::WhenAll(TLatentContext\<const UObject\> LatentContext, TAwaitable auto&&... Awaitables)

仅供在游戏线程上进行高级用法。
本函数的工作方式与普通 WhenAll 相同，但各参数会以 latent 模式、
使用传入的上下文被 await。

示例：
```cpp
using namespace UE5Coro::Latent;

co_await WhenAll(this, TaskA, TaskB);
```
