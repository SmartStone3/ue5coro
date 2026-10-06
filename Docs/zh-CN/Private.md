# 私有实现细节

[English](../Private.md) | 简体中文

> [!CAUTION]
> 本页介绍 UE5Coro 的部分内部工作原理，希望对制作 fork 或竞品插件的人有所帮助。
>
> 假定读者对 C++ 协程，以及语言中 coroutine_traits、coroutine_handle、
> promise_type 等的用法非常熟悉。
>
> **私有 API 中的一切都可能在任何版本中改变，且不会事先标记弃用。**

## 断言

由于 C++ 中的内存处理可能非常复杂，尤其是涉及异步执行时，
插件代码大量使用 `check` 以及其他旨在尽早发现 bug 的功能。
它们总是附带一条消息：
* `ensureMsgf` 表示你的做法很可能是错的，但已被处理。
* `checkf` 表示你的做法肯定是错的，并且很可能会崩溃。
* `checkf(..., "Internal error: ...")` 表示你发现了插件的 bug。

## 背景与历史

这个插件的设计并不完美，很可能也不是最优的。
它的关注点始终是便利性优先，面向 gameplay 以及与之相邻的、对性能不敏感的代码。
少数几个主要功能（主要是返回值和通用的取消）进一步增加了开销。

UE5Coro 0.9 从未公开发布；它最初是某个商业项目的一部分，那时还没有单独的名字。
在作为自由软件发布 1.0 版时，它经过了大幅重新设计，
统一了异步协程和 latent 协程可用的功能集。

从 1.0 起，改动在一定程度上受到保持与现有代码兼容的约束，尽管这并非铁定的承诺。
1.x 和 1.y 版本之间有过大量破坏性变更，这一点和 Unreal 本身差不多。

拥有两种协程 promise，以及各种 awaiter **不**共享一个公共基类，
或许是缺点最多的两个设计决策。
不过它们也并非一无是处，也带来了一些好处，这就是 2.0 仍在沿用它们的原因。

### FAsyncPromise/FLatentPromise

promise 会在[后文](#promises)更详细地介绍。
为协程提供两套底层实现，主要动机可以追溯到 UE5Coro 0.9：
当时它们是两套独立的系统，各有各的 awaiter，实现也非常直接、极简。

异步协程返回 FAsyncCoroutine，latent 协程返回 FLatentCoroutine，
而且它们只能 await 各自对应命名空间中的东西。

异步 awaiter 从回调中调用 `coroutine_handle::resume()`；
latent awaiter 锁定在游戏线程上，使用的是 2.0 中仍然存在的 FLatentAwaiter 系统。

用户最大的抱怨就是这种割裂，很快就清楚了：统一功能集势在必行。
主要的障碍是在 latent 协程中使用多线程，而它的所有权仍留在游戏线程上。
latent 动作管理器是引擎代码，它不会配合。

所有权问题由 DetachFromGameThread 解决，[后文](#flatentpromise)会介绍。
反方向的跨越则直接得多；它由 FPendingAsyncCoroutine
（与 FPendingLatentCoroutine 相对）处理。

#### 手动协程

手动协程使用 FAsyncPromise，但重写了它的 promise extras 类型，
以额外容纳一个 FAwaitableEvent 和第二个原子引用计数器。
这样做是为了避免在一个就够用的时候用两个 std::shared_ptr。

### 没有 awaiter 基类

这带来了一些相当明显的限制，例如 WhenAny/WhenAll 不能接受
TArray\<IAwaiter*\>：它们的类型和数量必须在编译期已知。

之所以选择这种权衡，主要有两个原因：一是性能（如下一节所述，这在当时更为重要），
二是有些类型会根据 await 它们的协程种类而表现不同。

### 内存管理

另一种设计可以直接解决上述所有限制；在插件早期曾探索过这种设计，
最终没有采用，但它被认为是完全可行的。
也许有一天，会出现一个走这条路或类似路线的竞品插件。

在这种设计中，协程 promise 和 awaiter 都是共享指针，
把所有内存管理问题都推迟到运行时处理。
latent 动作被 `delete` 也没什么大不了的，因为它只会拿走一个引用，
而协程 promise 大概率仍然存活。
引擎回调可以接收指向 awaiter 的弱指针，
从而把永不结束的 await 造成的内存泄漏缩小到只泄漏控制块。

这大致就是 .NET 等环境中协程的实现方式，但在 C++ 中它会带来惊人的性能损失。
1.7 版加入 `FPromiseExtras`（它保存在一个共享指针中）时，
协程创建时间跃升了 30%！
由于现在多线程支持已无处不在，对 UE5Coro 来说，
使用线程不安全的 TSharedPtr 不是一个选项。

.NET 的处境更好，因为它的内存分配比 C++ 快，传递对象也不涉及原子引用计数。

当初统一 FAsyncPromise/FLatentPromise 时，这些额外开销被认为太高；
但为了支持协程返回值，它又是必需的——返回值带来的影响要大得多，
被认为值得付出这些开销。

当然，这意味着现在既然已经有了 `FPromiseExtras`，
更多地使用共享指针所带来的额外开销，按比例来说会比当时更低。

## 调试工具

Debug.h 包含协程追踪，以及几个未使用、但可能有用的调试工具。

把 `UE5CORO_PRIVATE_USE_DEBUG_ALLOCATOR` 定义为 1，
会只对协程状态应用一个轻量版的 stompmalloc（仅限 Windows）。
它会有意泄漏地址空间（内存被解除提交，但不释放），
以强制新的 promise 分配在不同的地址上。

ClearEvents/GEventLog 可以用作低开销的事件记录器，用于调试多线程问题。
在出问题的代码段之前调用 ClearEvents()，
再用 `UE5CORO_PRIVATE_DEBUG_EVENT(几乎任何东西)` 来追踪执行。
可以把 `bLogThread` 设为 true 来记录每条消息的来源线程，但开销会更高。

`GLastDebugID`、`GActiveCoroutines` 和 `GPromises`（如果启用了协程追踪）
可以帮助追查协程泄漏。
如果启用了协程追踪，用 [Gameplay Debugger](GameplayDebugger.md) 会更方便。
`FPromiseExtras::DebugID` 与 `GLastDebugID` 使用同一个计数器。

在某些编译器上，`Use()` 可以用来延长某些基本类型局部变量的生命周期，
这些变量即使在调试构建中也会被优化掉。

## 代码风格

虽然 UE5Coro 力求提供一个原生的、具有 Unreal 风格的公共 API，
但在被认为有益的地方，它的实现偏离了传统的、“纯粹”的 Unreal 风格。

### 使用 STL

凡是 STL 类型的表现与 Unreal 重新发明的版本相当或更好的地方，就选用了 STL 类型；
还有少数情况下，使用 STL 是为了避开 Unreal 版本中的 bug。
可惜的是，后一类中的部分用法也出现在了公共 API 中，
以避免不必要的封装或转换。

### 移动语义

有几个函数按值接受“昂贵”的参数，而不是更常用的 const 引用，
例如用 TArray 而不是 const TArray&。

这是按具体情况决定的：参数需要更长的生命周期，或者会以其他方式被消耗掉。
这样调用方就可以通过传入右值来避免一次拷贝。
如果使用 const 引用，即使调用方可以选择移动数据，也会被迫进行拷贝。
Unreal 自己的代码经常犯这个错误。

优先使用 std::move 和 std::forward，而不是它们的 Unreal 对应物，
因为在 MSVC 下它们是编译器内建函数，在调试构建中性能更好。

## 取消

取消功能长期以来呼声很高，而最初被认为几乎不可能实现。
背后有大量工程投入的主流系统基本上都放弃了这一点，
转而期望协程自己实现协作式取消（例如 .NET 中的 CancellationToken）。

UE5Coro 的关键领悟在于：与其去解决那个通用的、几乎（或者真的？）不可能解决的取消问题，
不如提供一个大大简化的版本，它只会在协程**没有**运行时发生。

这就是为什么取消只在 `co_await` 时处理：此时协程处于一个已知的、可以安全转向的状态。

协程之所以在恢复时处理取消，是因为许多引擎函数会复制传入的委托，
而它们自身又不提供任何取消支持。
要处理这些情况，需要类似前面提到的“一切皆共享指针”的方案，开销会更高。

尽管如此，在被认为重要的地方，有些 awaiter 确实使用了共享指针。
UE5Coro::Private::FTwoLives 实现了它的一个简化版本，
用于恰好有两方共享数据的情况：awaiter，以及被封装的那个引擎函数。
它只有一个指针大小，其中已经包含了 32 个可用于自定义数据的空闲位。

能够处理加速取消的 awaiter，可以通过调用
FPromise::RegisterCancelableAwaiter 来声明支持。

<a id="promises"></a>
## Promise

std::coroutine_traits 的内置特化会检查实参，
寻找 FLatentActionInfo、FForceLatentCoroutine 和 TLatentContext 参数，
以确定执行模式，并据此参数化 TCoroutinePromise。

涉及的类有（都在 `UE5Coro::Private` 命名空间中）：

* `FPromise` 实现取消、ContinueWith 以及透传的异常处理。
  在 C++ 中，即使关闭了异常，协程 promise 也必须对未处理的异常做出反应。
* `FPromiseExtras` 包含生命周期与 FPromise 不一致的字段，例如协程结果的存储。
  TCoroutine 和 FPromise 都持有指向它的 shared_ptr。
* `FAsyncPromise` 为异步模式提供简单的实现，异步模式总体上是两种执行模式中较简单的一种。
* `FLatentPromise` 包含与 latent 动作管理器之间转移所有权的逻辑。
* `TCoroutinePromise<T, Base>` 继承自上面两种 promise 之一，并增加了返回类型支持，
  对 `void` 有一个特化。<br>
  `TCoroutine` 通常以它作为 `promise_type`。
* UE5CoroGAS 有一个特殊的 `TAbilityPromise`，它以所属的类而不是返回类型作为参数。

### FPromise

可以在协程体中调用 `FPromise::Current()`，从任何线程访问当前的 promise。
与取消相关的类型在内部用它来直接与 promise 通信。

在 FPromise 上调用 `get_return_object()`，可以让协程在返回值被返回之前就拿到它，
这有时很有用。

在调试构建中还会存储一些额外的调试数据，并支持 natvis。

### FAsyncPromise

这个类大体上平平无奇，提到它只是为了完整起见。
使用这种 promise 的协程，其执行方式与“纯” C++ 协程大体相似，
主要处理的是回调，而不是 Unreal 风格的轮询/tick。

### FLatentPromise

这个类在 latent TCoroutine 与其背后的 latent 动作之间架起桥梁。

在 initial_suspend 中，它模仿通常的蓝图行为：如果已经有另一个相似的 latent 动作
正在运行（比如在 Delay 节点处于激活状态时又执行到它），就**不**开始执行；
否则它会创建一个 `new FPendingLatentCoroutine` 并将其注册到 world 中。
world 拥有这个对象，并间接拥有协程的执行，但这条规则有一个值得注意的复杂例外。

UE5Coro 的标志性特性之一是：蓝图/latent 功能与完整的异步多线程功能
在两种模式下都能无缝工作，可用的功能完全相同
（只有少数与背后 latent 动作交互相关的 latent 专属功能，它们对异步模式没有意义）。

这与 latent 动作管理器在游戏线程上**拥有**协程执行这一点直接冲突。
有可能在协程正在后台线程上执行时，
latent 动作管理器在游戏线程上 `delete` 了它的 pending latent action。

FLatentPromise 能够从游戏线程“分离”（detach）。
在这种状态下，它会暂时接管协程执行的所有权，
确保协程可以安全地到达下一个 co_await，在那里检查所有到来的 `delete`，
并在需要时处理它们。
大多数分离操作由 TAwaiter::await_suspend 自动完成，
派生类不应重载它（如果 C++ 允许的话，它会被声明为 `final`），而应改用 `Suspend`。

任何分离 FLatentPromise 的 awaiter 都必须保证在某个时刻调用 FPromise::Resume，
它会决定所有权是应该回到游戏线程（以及 latent 动作管理器），还是保持分离。

到达 final_suspend 的 latent 协程，总是会把 promise 重新附加到游戏线程上。

## Awaiter

虽然所有 awaiter 都能让 C++ 满意，但其中许多并不能正常工作。
例如，`co_await std::suspend_always();` 能够编译，但使用时会泄漏内存，
因为没有公开的方式可以直接恢复 coroutine_handle
（最接近这项功能的是 UE5Coro::FAwaitableEvent）。

至关重要的是调用 promise 的 Resume() 函数，而不是 coroutine_handle::resume()。
这让 promise 可以处理取消，对于 FLatentPromise 还可以处理游戏线程所有权。

本插件中的大多数 awaiter 继承自两种基类型之一：用于游戏线程轮询的 FLatentAwaiter，
以及用于其他所有需要分离 FLatentPromise 的情况的 TAwaiter/TCancelableAwaiter。
它们被设计为不使用任何 `virtual`，不过 FLatentAwaiter 和 TCancelableAwaiter
内部各有一个普通的函数指针（比虚表少一层间接）。

使用它们，你就可以定义自己的 awaiter 类型，并让它们像内置的那样工作，
而不必依赖公共 API 上的通用 awaiter
（TCoroutine\<\>、委托、FAwaitableEvent、Latent::Until……）。

### TAwaiter

这个小小辅助类的主要功能，是把 FLatentPromise 从游戏线程上分离，
并把 await_suspend(std::coroutine_handle\<T\>) 调用转换为 Suspend(T&)，
从而可以针对 promise 类型进行重载和多态。
它还为 await_ready（未就绪）和 await_resume（void 空操作）提供了合理的默认实现。

它使用 CRTP 和静态分派，以确保尽可能多的代码可以被内联。
awaiter 可以实现 `Suspend(FPromise&)`，
也可以实现 `Suspend(FAsyncPromise&)`+`Suspend(FLatentPromise&)`，
这样就能覆盖所有 TCoroutinePromise 类型。
进一步的继承层级要非常小心。
通常这样做是为了提供更好的 await_resume 实现，用于静态分派。
在其他场景下，你可能需要考虑再加一层 CRTP，让 T 被设为最终派生的子类。

许多 awaiter 假定 await_suspend 会在 await_ready 返回 false 之后立即被调用，
并为了额外的安全性和性能而让锁保持锁定，
而不是让 await_ready 保持 const、在 await_suspend 中再次加锁、然后重新检查。

其中一些本可以是 const 的，但有些 await_ready 确实会修改状态。

### TCancelableAwaiter

这个 TAwaiter 子类用于支持直接加速取消的 awaiter。
这涉及处理一个复杂的多线程场景，最多有三个线程同时竞争
（一个从游戏线程分离的 latent 协程在游戏线程上被销毁、在线程 A 上被恢复、
在线程 B 上被取消）。

不要在派生类型中遮蔽 `fn_`。
它的命名不同寻常，就是为了降低这种情况发生的可能性。

#### 如何编写 TCancelableAwaiter

在 Suspend() 中：
* 锁定 promise。
* 调用 `Promise.RegisterCancelableAwaiter(this)`。
* 如果它返回 true，Cancel() 可能在任何时刻被调用，包括就在此刻，
  但它会阻塞直到 promise 被解锁。
* 如果它返回 false，按需清理，然后无条件地**异步** `Resume()` 协程。
* 只恢复 UnregisterCancelableAwaiter() 返回 true 的 promise。
  正常的恢复允许是同步的。

在 `fn_`（Cancel()）中：
* 你已经持有 Extras->Lock，它会阻塞可能发生的 ~FPromise()
  以及 UnregisterCancelableAwaiter\<true\>()。
* 调用 `Promise.UnregisterCancelableAwaiter<false>()`。
  如果它返回 false，什么也不做。
  说明另有一方得到了 true，并已安排好恢复协程。
* 如果它返回 true，按需清理，然后**异步**地 `Resume()` 协程。

awaiter 必须保证取消与 Suspend/Resume 之间的线程安全，
并保证恰好恢复协程一次。

通常，这是 UnregisterCancelableAwaiter() 除第一次之外每次都返回 false 的自然结果，
但也可能需要额外的同步。
如果 awaiter 对象是局部变量或临时对象，销毁协程就会销毁它。
恢复一个已被取消的协程，可能会导致它（以及 awaiter）在 Resume() 返回之前就被销毁。

### FLatentAwaiter

这个基类用于在游戏线程上基于轮询/tick 的 awaiter。
它**不是**从 TAwaiter 派生的，它的 Suspend() 实现会根据执行 await 的 promise 而大不相同。

虽然每种 latent awaiter 类型都被视为私有的，但它们是 latent awaiter 这一事实
通过 `UE5Coro::TLatentAwaiter` 概念对外暴露，主要是为了文档目的；
不过一些刻意设计的模板代码也可以据此推断任意 awaiter 的预期行为。

State 是一个通用指针，但许多 latent awaiter 把它当作 64 位存储空间来用，
以避免间接访问和/或额外的堆分配。
例如，一个 `double` 正好放得下。
这一假设由一个 static_assert 守护，不过 Unreal Engine 5 本身反正也不支持 32 位平台。

#### 异步/Latent

用 FAsyncPromise await 一个 FLatentAwaiter，会在 world 中创建一个 latent 动作来 tick 该 awaiter。
与 latent 协程不同，异步协程会为每一次单独的 co_await 创建一个 latent 动作。

这意味着 latent 动作管理器会暂时拥有一个通常自己拥有并管理自己的协程。

latent 动作被 `delete` 会被转换为一次普通取消。
由于异步协程拥有自己，这次取消不是强制的。
被 `delete` 的 Private::FPendingAsyncCoroutine 会立即交还所有权，
而不是把协程一起带走。

#### Latent/Latent

FLatentPromise 有一条 co_await FLatentPromise 的快速路径：
它会被透传给底层的 FPendingLatentCoroutine 并直接在那里轮询，
在其活跃期间绕过 promise 和协程，直接与 latent 动作管理器通信。

这种行为还让这些 awaiter（并且_只有_这些 awaiter）能够在下一个 tick
就对到来的取消做出反应，而不必等待某个回调发生，
才能在一个可能晚得多的时间点正确地清理并处理取消。

继承 FLatentAwaiter 必须小心：派生对象在被复制时会发生对象切片，
所以它的 `sizeof` 必须保持不变。
通常这样做只是为了重新实现 `await_resume`，
根据 State 字段从 await 表达式返回一个有意义的值。

FLatentAwaiter 部分会被复制到 promise 中，以尽可能消除间接访问。
UE5Coro 1.x 使用的是直接指向 co_await 表达式的指针，这允许多态，
但实践中从未需要过多态。
这也是为什么 `Resume` 是函数指针而不是虚方法：只有一个条目的虚表会多一层间接。

如果不需要返回值，定义新的 latent awaiter 相当简单：
返回 FLatentAwaiter 本身，用任意的状态和一个合适的函数指针初始化它，
该函数会在 co_await 时调用一次，之后每个 tick 调用一次。
如果函数对 world 的变化敏感，第三个构造参数应为 std::true_type()，
否则为 std::false_type()。
这只影响调试构建中的一个 ensure()。

如果协程应当被恢复，就返回 true；如果 bool 参数为 true，就清理状态。
请参考插件代码中大量现成的示例。
这个内部类型在插件的整个开发过程中一直相当稳定、一致。

以 bCleanup 为 true 调用时，Resume 的返回值会被忽略。

## TAwaitTransform

这个 trait 允许通过特化，从任何地方（包括插件模块之外的客户端代码）
扩展 promise 的 `await_transform`。
默认实现会原样转发其参数；如果参数有 `operator co_await`，则调用它。

这是让 Unreal 类型无需封装函数或 reinterpret_cast 技巧就可以被 await 的主要方法，
不过后者仍被用于提供更高效的右值实现。

TAwaitTransform 会收到它正在为之做转换的 promise 种类。
TCoroutine 正是借此，根据在哪种协程中被 await，
分别创建基于 ContinueWith 的 TAsyncCoroutineAwaiter，
或基于 FLatentAwaiter 的 TLatentCoroutineAwaiter。

## 蓝图支持

协程 UFUNCTION 的调用由一个自定义 K2Node 处理，
目的是去掉无用的引脚，让蓝图看起来清爽一些。
这些改动纯粹是视觉上的。

它的工作原理是在运行时修补 UFunction 本身，这样你就不必亲手把每一个协程
都标记为 BlueprintInternalUseOnly。
这在 Shipping 构建中没有任何性能损失，因为 K2Node 只存在于编辑器中。

它的提示文字也被改成了 "Call Coroutine"，这样如果出于某种原因需要，
在编辑器里很容易把它区分出来。
它是一个 LOCTEXT，以防你的团队使用本地化的编辑器。

如果你出于某种原因不喜欢它，在把 UE5Coro 引入项目时删掉这个类非常简单，
一切照常工作。
之后再删掉它，则需要添加一条重定向回 K2Node_CallFunction 的核心重定向。
