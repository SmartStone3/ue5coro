# UE5Coro

[English](../README.md) | 简体中文

> [!NOTE]
> 本文是 [UE5Coro](https://github.com/landelare/ue5coro) 英文文档的非官方中文译文，
> 如有出入以[英文原文](../README.md)为准。版权与许可见 [COPYING](../COPYING)。

UE5Coro 为 Unreal Engine 5 实现了 C++20
[协程](https://en.cppreference.com/w/cpp/language/coroutines)支持，着重于
gameplay 逻辑、易用性，以及与引擎的无缝集成。

插件内置了对 latent（延迟）UFUNCTION 的便捷编写支持。
只需把 latent UFUNCTION 的返回类型改掉，它就成了协程，所有
FPendingLatentAction 样板代码全部免费奉送，并且开箱即用地支持蓝图安全的多线程：
```cpp
UFUNCTION(BlueprintCallable, meta = (Latent, LatentInfo = LatentInfo))
FVoidCoroutine Example(FLatentActionInfo LatentInfo)
{
    UE_LOGFMT(LogTemp, Display, "Before delay");
    co_await UE5Coro::Latent::Seconds(1); // 不会阻塞游戏线程！
    UE_LOGFMT(LogTemp, Display, "After delay");

    // 离开游戏线程就是这么简单……
    co_await UE5Coro::Async::MoveToTask();
    UE_LOGFMT(LogTemp, Display, "In game thread: {0}", IsInGameThread());
    FString Value = TEXT("Imagine this was expensive to compute");

    // ……回到游戏线程也一样：
    co_await UE5Coro::Async::MoveToGameThread();
    UE_LOGFMT(LogTemp, Display, "In game thread: {0}", IsInGameThread());
    UE_LOGFMT(LogTemp, Display, "Value: {0}", Value);
}
```

这是一个真实可运行的示例！
试着把它放进一个 actor 类里。
甚至可以在协程运行期间销毁这个 actor。
latent 协程会自动追踪它的目标 UObject，必要时提前清理。

就连协程的返回类型也对蓝图隐藏了，不会打扰到策划：

![上面 Example 函数对应的 latent 蓝图节点](../Docs/latent_node.png)

对 latent UFUNCTION 不感兴趣？
没关系。
同样支持纯 C++ 协程，功能完全一致。
底层实现在编译期选择；只有真正用到时才会创建 latent 动作。

把返回类型改成本插件提供的某个协程类型，那些自己实现起来很繁琐的复杂异步任务，
就会变成一行搞定、Just Work™ 的代码，再也不需要回调和其他处理函数。

* 喜欢 LoadSynchronous 的方便，却不想要它的缺点？<br>
  `UObject* HardPtr = co_await AsyncLoadObject(SoftPtr);` 让你只保留好处（还有你的帧率）。
* 想把一段繁重的计算分摊到多个 tick 上？<br>
  在循环里加一句 `co_await NextTick();` 就完事了。
  还有一个时间预算类，可以直接指定期望的处理时长，让协程动态地自我调度。
* 说到动态调度，节流可以简单到这样：<br>
  `co_await Ticks(bCloseToCamera ? 1 : 2);`
* 明明有好几个 CPU 核心跃跃欲试，为什么还要在游戏线程上做时间切片？<br>
  在函数里加一句 `co_await MoveToTask();`，这一行之后的所有代码都会在
  UE\:\:Tasks 系统的工作线程上运行。<br>
* 想回去？
  那就 `co_await MoveToGameThread();`。<br>
  你可以在线程之间随意切换，latent UFUNCTION 协程结束时会自动回到游戏线程，
  以便恢复蓝图的执行。
* 还没被说服？下面是运行一整条时间轴的写法：<br>
  `co_await Timeline(this, From, To, Duration, YourUpdateLambdaGoesHere);`
* 很难讨好？
  下面是异步等待一个 DYNAMIC 委托的写法，不用专门为 AddDynamic/BindDynamic
  写一个 UFUNCTION，不用身处 UCLASS 中，甚至不用身处任何类中：<br>
  `co_await YourDynamicDelegate;`（这就是全部代码）
* 哦，你还想要委托的参数？<br>
  `auto [Your, Parameters] = co_await YourDynamicDelegate;`

以上应该能让你体会到，这个插件可以大幅减少代码量和工作量。
要写的代码更少、更简单，通常意味着 bug 更少；而异步代码写起来容易，
意味着一开始就用正确的方式做事毫无阻力。

跟那个“暂时够用™”、两个版本之后还留在 Shipping 里的 LoadSynchronous 说再见吧。
当手头的任务从“写一堆 StreamableManager 样板代码，再把调用函数的一部分挪进回调”
简化成仅仅“在前面加个 co_await”，你完成它所花的时间，
比想出一个理由来解释为什么同步阻塞版本还能接受还要短。

插件里还有很多其他功能，比如生成器（generator），让你不必为了返回数量不定的值
而分配一整个 TArray。
你写起来更容易，编译器也更容易
[优化](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1365r0.pdf)，
N 个元素只需要 O(1) 而不是 O(N) 的存储，有什么理由不喜欢呢？

# 功能列表

下面的链接会带你前往文档的相应页面。
如果你喜欢在浏览器里阅读最新文档，可以收藏本节；也可以直接在 IDE 里阅读。
每个 API 函数都在 C++ 头文件中有文档注释，发行版中也包含你现在正在阅读的
这份文档的 Markdown 源文件。

## 编写协程

这些功能侧重于把协程暴露给引擎的其他部分。

* [协程](../Docs/zh-CN/Coroutine.md)（显而易见）
  * [取消](../Docs/zh-CN/Cancellation.md)支持有单独的页面。
* [生成器](../Docs/zh-CN/Generator.md)
* [Gameplay Ability System](../Docs/zh-CN/GAS.md) 集成的工作方式略有不同。
<!-- 另有一个未列出的文档页面：Private.md -->

## Unreal 集成

这些封装让你可以在协程中方便地使用引擎功能。

* [AI](../Docs/zh-CN/AI.md) 集成（MoveTo、寻路……）
* [动画 awaiter](../Docs/zh-CN/Animation.md)（蒙太奇、通知……）
* [异步 awaiter](../Docs/zh-CN/Async.md)（多线程、同步……）
  * [异步链](../Docs/zh-CN/AsyncChain.md)（适用于接受委托参数的函数的通用封装）
* [HTTP](../Docs/zh-CN/Http.md)（异步 HTTP 请求）
* [隐式 awaiter](../Docs/zh-CN/Implicit.md)（某些引擎类型无需封装即可直接 co_await）
* [Latent awaiter](../Docs/zh-CN/Latent.md)（与游戏线程交互、Delay……）
  * [资产加载](../Docs/zh-CN/LatentLoad.md)（异步加载软指针、资产包……）
  * [异步碰撞查询](../Docs/zh-CN/LatentCollision.md)（射线检测、重叠检测……）
  * [Latent 链](../Docs/zh-CN/LatentChain.md)（通用的 latent 动作封装）
  * [Tick 时间预算](../Docs/zh-CN/LatentTickTimeBudget.md)（每帧运行 x 毫秒）
* [Latent 回调](../Docs/zh-CN/LatentCallback.md)（与 latent 动作管理器交互）

> [!NOTE]
> 这些函数大多返回 `UE5Coro::Private` 命名空间中未写入文档的内部类型。
> 客户端代码不应直接引用该命名空间中的任何东西，因为其中的一切都可能在未来版本中
> 改变，并且**不会**事先标记弃用。
>
> 大多数情况下这不成问题：例如 `co_await Something()` 中的匿名临时对象
> 根本不会出现在源代码里。
> 如果需要保存一个 `Private` 返回值，请使用 `auto`（或带约束的
> `TAwaitable auto`），避免写出该类型的名字。
>
> 不支持在任何 awaiter 上直接调用那些出于需要而公开的 C++ awaitable 函数
> `await_ready`、`await_suspend` 和 `await_resume`。

## 其他功能

* [聚合 awaiter](../Docs/zh-CN/Aggregate.md)（WhenAny、WhenAll、Race……）
* [Latent 时间轴](../Docs/zh-CN/LatentTimeline.md)（在 tick 上平滑插值）
* [线程原语](../Docs/zh-CN/Threading.md)（信号量、事件……）
* [Gameplay Debugger](../Docs/zh-CN/GameplayDebugger.md) 集成以及
  [本地化](../Docs/zh-CN/GameplayDebugger.md#the-conditional-modifier)工具

# 安装

只支持带编号的发行版（release）。
不要直接使用 Git 分支。

下载你选定的发行版，解压到项目的 Plugins 文件夹。
把文件夹重命名为不带版本号的 UE5Coro。
操作正确的话，最终应得到
`YourProject\Plugins\UE5Coro\UE5Coro.uplugin`。

## 项目设置

你的项目可能使用了一些旧版设置，需要移除它们才能启用 C++20 支持；
在 Unreal Engine 5.3 及以后新建的项目中，C++20 本来就是标配。

在你的 **Target.cs** 文件（所有的）中，确保使用的是最新的构建设置和 include 顺序版本：

```c#
DefaultBuildSettings = BuildSettingsVersion.Latest;
IncludeOrderVersion = EngineIncludeOrderVersion.Latest;
```

如果你在使用旧版的 `bEnableCppCoroutinesForEvaluation` 标志，现在已经不需要它了，
也不应再显式开启它；开启可能会导致问题。
建议从构建文件中移除所有对它的引用。

如果你在某个 Build.cs 里把 `CppStandard` 设成了 `CppStandardVersion.Cpp17`……
别这么做 :)

## 使用

像引用其他 C++ 模块一样，在 Build.cs 中引用 `"UE5Coro"` 模块，并使用
`#include "UE5Coro.h"`。
插件本身不需要启用。

部分功能位于需要单独引用的可选模块中。
例如，Gameplay Ability System 支持需要在 Build.cs 中加入 `"UE5CoroGAS"`，
并 `#include "UE5CoroGAS.h"`。
核心 UE5Coro 模块只依赖默认启用的引擎模块。

> [!IMPORTANT]
> 不要直接 #include 其他任何头文件，只 include 与模块同名的那一个。
> 众所周知，与 Unreal Engine 搭配使用的主流 IDE 会给出错误的头文件建议。
> 如果把 UE5Coro.h 加入你的 PCH，就可以在所有地方使用它。

# 更新

更新时，先从项目的 Plugins 文件夹中删除 UE5Coro，再按照上面的说明安装新版本。

# 打包

不需要、也不支持单独打包 UE5Coro（通过 Plugins 窗口）。

# 移除

要从项目中移除本插件：在不使用其功能的前提下重新实现你所有的协程，
移除对本插件及其模块的所有引用，并添加一条从
`/Script/UE5CoroK2.K2Node_UE5CoroCallCoroutine` 到
`/Script/BlueprintGraph.K2Node_CallFunction` 的核心重定向（core redirect）。
