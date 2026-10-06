# Tick 时间预算

[English](../LatentTickTimeBudget.md) | 简体中文

`UE5Coro::Latent::FTickTimeBudget` 提供了一种方便的方式，把游戏线程上的处理
限制在指定的时长内，而不必手动调整每个 tick 处理多少个元素[^timeslice]。

[^timeslice]: 后者在循环里用
              `if (i % LoopsPerTick == 0) co_await NextTick();` 就能轻松实现。

这个类不算 latent awaiter（它不是某个函数返回的未写入文档的类型），
但它只能在游戏线程上使用。

在预算耗尽之前，await 它什么也不做；预算耗尽后，它会像
`UE5Coro::Latent::NextTick()` 那样表现一次，然后又恢复为一段时间内什么也不做。

这个类型的预期用法是：在循环外创建，然后在循环内反复 await。
这种模式保证无论耗时多久，至少会运行一次迭代。

> [!TIP]
> 虽然异步协程和 latent 协程都可以使用这项功能，但它针对 latent 协程做了优化。
> 异步协程的额外开销是每个 tick 一个固定量，与一个 tick 内能容纳多少次 co_await 无关。

### static FTickTimeBudget Seconds(double SecondsPerTick)

返回一个对象，每个 tick 放行指定秒数的代码执行时间。

### static FTickTimeBudget Milliseconds(double MillisecondsPerTick)

返回一个对象，每个 tick 放行指定毫秒数的代码执行时间。

### static FTickTimeBudget Microseconds(double MicrosecondsPerTick)

返回一个对象，每个 tick 放行指定微秒数的代码执行时间。

## 示例

以 1 毫秒的预算处理固定数量的元素：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Latent;

TCoroutine<> ProcessItems(TArray<FExampleItem> Items, FForceLatentCoroutine = {})
{
    auto Budget = FTickTimeBudget::Milliseconds(1);
    for (auto& Item : Items)
    {
        ProcessItem(Item);
        co_await Budget;
    }
}
```

多阶段处理同样适用，这样可以提高把工作推迟到下一 tick 的粒度。
协程会从上次离开的地方继续：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Latent;

TCoroutine<> ProcessItems(TArray<FExampleItem> Items, FForceLatentCoroutine = {})
{
    auto Budget = FTickTimeBudget::Milliseconds(1);
    for (auto& Item : Items)
    {
        PreProcess(Item);
        co_await Budget;
        ProcessCore(Item);
        co_await Budget;
        PostProcess(Item);
        co_await Budget;
    }
}
```

处理其他线程发送到游戏线程的元素（例如生成 actor 的指令），每个 tick 允许 2 毫秒：
```cpp
using namespace UE5Coro;
using namespace UE5Coro::Latent;

TCoroutine<> ProcessItems(TMpscQueue<FExample>& Queue, FForceLatentCoroutine = {})
{
    for (;;)
    {
        auto Budget = FTickTimeBudget::Milliseconds(2);
        for (FExample Item; Queue.Dequeue(Item); co_await Budget)
            ProcessItem(Item);
        // 如果队列为空，推迟到下一个 tick，否则外层循环会卡死游戏线程
        co_await NextTick();
    }
}
```

每个 tick 反复调用一个工作函数 0.5 毫秒（500 微秒）：
```cpp
using namespace UE5Coro::Latent;

for (auto Budget = FFrameTimeBudget::Microseconds(500); !bDone; co_await Budget)
    PerformOneStepOfWork();
```
