# UE5CoroAI

[English](../AI.md) | 简体中文

这个可选的附加模块与引擎“经典”的 AI 功能集成，例如 AIModule、NavigationSystem，
但不包括 MASS。
它为借助这些系统执行的各种任务（例如“AI Move To”）提供了 awaiter。

不使用这些系统的项目不会从本模块中受益。

> [!IMPORTANT]
> 本模块目前处于 **beta** 阶段。
>
> 未来版本中如果出于修复需要，可能会出现破坏性的行为变更，
> 不过 API 本身打算保持稳定。

要使用 UE5CoroAI，请在 Build.cs 中引用 "UE5CoroAI"，并使用
`#include "UE5CoroAI.h"`。

本页介绍的所有内容都位于 `UE5Coro::AI` 命名空间中。
更多细节请参考这些函数声明上方的注释，以及它们所封装的引擎函数。
该命名空间中的所有函数和 awaiter 都只能在游戏线程上使用。

### auto FindPath(UObject* WorldContextObject, const FPathFindingQuery& Query, EPathFindingMode::Type Mode)

使用 `UNavigationSystemV1::FindPathAsync` 发起一次异步寻路操作。

await 本函数返回的对象，结果是一个包含寻路结果的
`TTuple<ENavigationQueryResult::Type, FNavPathSharedPtr>`。

返回的 awaiter 是世界敏感（world sensitive）的，详见[此页](Latent.md)。

示例：
```cpp
using namespace UE5Coro::AI;

if (auto [Result, Path] = co_await FindPath(this, Query, Mode);
    Result == ENavigationQueryResult::Success)
    DoSomethingWith(Path);
```

### auto AIMoveTo(AAIController* Controller, ? Target, float AcceptanceRadius = -1, EAIOptionFlag::Type StopOnOverlap = EAIOptionFlag::Default, EAIOptionFlag::Type AcceptPartialPath = EAIOptionFlag::Default, bool bUsePathfinding = true, bool bLockAILogic = true, bool bUseContinuousGoalTracking = false, EAIOptionFlag::Type ProjectGoalOnNavigation = EAIOptionFlag::Default)

本函数有两个重载：`Target` 可以是 AActor* 或 FVector。

它会用传入的参数调用 `UAITask_MoveTo::AIMoveTo`。

await 返回值后，移动无论因何结束（包括失败），协程都会恢复执行。
await 表达式会给出 EPathFollowingResult。

返回的 awaiter 是世界敏感的，详见[此页](Latent.md)。

示例：
```cpp
using namespace UE5Coro::AI;

AActor* Target = GetTarget();
if (co_await AIMoveTo(Controller, Target) == EPathFollowingResult::Success)
    HandOverItem(Target, Item);
```

### auto SimpleMoveTo(AController* Controller, AActor* Target)
### auto SimpleMoveTo(AController* Controller, FVector Target)

这两个函数的行为与
`UAIBlueprintHelperLibrary::SimpleMoveToActor`（AActor* 重载）或
`UAIBlueprintHelperLibrary::SimpleMoveToLocation`（FVector 重载）完全相同，
包括它们内部写死的常量和其他怪癖（例如往控制器里注入一个组件），
并返回一个对象，await 它即可异步拿到移动的结果。

本函数不要求控制器是 AI 控制器。

await 返回值后，移动无论因何结束（包括失败），协程都会恢复执行。
await 表达式的结果是该操作的 FPathFollowingResult。

返回的 awaiter 是世界敏感的，详见[此页](Latent.md)。

示例：
```cpp
using namespace UE5Coro::AI;

if (FPathFollowingResult Result = co_await SimpleMoveTo(Controller, Target);
    Result.IsSuccess())
    ArrivedAtTarget.Broadcast();
```
