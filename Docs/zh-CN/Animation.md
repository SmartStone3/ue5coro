# 动画 awaiter

[English](../Animation.md) | 简体中文

这些函数位于 `UE5Coro::Anim` 命名空间，让你可以在游戏线程上与蒙太奇及其通知交互。
对于需要对动画过程中发生的事情、或者对动画本身的变化做出反应的 gameplay 任务，
它们用起来很方便。

这些函数的返回值可以复制，但同一时刻只能 await 其中一个副本
（就好像它们是指向同一个东西的共享指针）。

不要重复使用返回值。
后续的 await 不保证返回有效的值。

### auto MontageBlendingOut(UAnimInstance* Instance, UAnimMontage* Montage)
### auto MontageEnded(UAnimInstance* Instance, UAnimMontage* Montage)

这两个函数等待蒙太奇在指定实例上开始混出（blend out）或结束。
await 它们会返回一个 bool，表示这是否由中断引起。

动画实例被销毁也算作中断。
如有需要，可以在 co_await 之后用 IsValid(AnimInstance) 单独处理这种情况。

示例：
```cpp
using namespace UE5Coro::Anim;

TCoroutine<> AActress::Dash()
{
    UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
    AnimInstance->Montage_Play(DashMontage);
    bool bInterrupted = co_await MontageBlendingOut(AnimInstance, DashMontage);
    // 动画结束后做点什么
}
```

### auto NextNotify(UAnimInstance* Instance, FName NotifyName)

等待指定实例上发生该名称的动画通知，也就是 AnimNotify_MyNotifyName()
会被调用的时机，但不需要实现一个带魔法名字的 UFUNCTION。

示例：
```cpp
using namespace UE5Coro::Anim;

TCoroutine<> AActress::Attack()
{
    UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
    AnimInstance->Montage_Play(AttackMontage);
    co_await NextNotify(AnimInstance, "AttackHit");
    // 命中时做点什么
}
```

### auto PlayMontageNotifyBegin(UAnimInstance* Instance, UAnimMontage* Montage)
### auto PlayMontageNotifyBegin(UAnimInstance* Instance, UAnimMontage* Montage, FName NotifyName)
### auto PlayMontageNotifyEnd(UAnimInstance* Instance, UAnimMontage* Montage)
### auto PlayMontageNotifyEnd(UAnimInstance* Instance, UAnimMontage* Montage, FName NotifyName)

这些函数等待给定蒙太奇的任意一个（2 参数重载）或指定的（3 参数重载）
play montage notify 在指定实例上开始或结束。
Play montage notify 与动画蒙太奇通知（animation montage notify）不是一回事。
它们与蒙太奇中的分支点（branching point）相关。

await 这些函数的返回值，3 参数重载的结果是 `const FBranchingPointNotifyPayload*`，
2 参数重载的结果是 `TTuple<FName, const FBranchingPointNotifyPayload*>`，
用于标识刚刚开始或结束的是哪个 play montage notify。

该指针指向引擎管理的内存，因此它的生命周期有限，只到下一次 co_await 或 co_return 为止。
它也可能是 nullptr。

示例：
```cpp
using namespace UE5Coro::Anim;

auto [Name, Payload] = co_await PlayMontageNotifyBegin(AnimInstance, Montage);
if (Payload)
{
    ExampleBeginHandler(Payload);
    // 旧的 Payload 值会变成悬空指针，但它立刻就被一个新指针替换了：
    Payload = co_await PlayMontageNotifyEnd(AnimInstance, Montage, Name);
    if (Payload)
    {
        ExampleEndHandler(Payload);
        co_await NextTick();
        // 此处 Payload 是一个非空的悬空指针。不要使用！
    }
}
```
