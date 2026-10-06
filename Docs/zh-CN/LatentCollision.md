# 异步碰撞查询

[English](../LatentCollision.md) | 简体中文

这些函数位于 `UE5Coro::Latent` 命名空间，返回的类型满足 TLatentAwaiter 概念，
因此适用 [latent](Latent.md#latent-awaiters) 规则。

### auto AsyncLineTraceByChannel(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, ECollisionChannel TraceChannel, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam, const FCollisionResponseParams& ResponseParam = FCollisionResponseParams::DefaultResponseParam)
### auto AsyncLineTraceByObjectType(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, const FCollisionObjectQueryParams& ObjectQueryParams, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)
### auto AsyncLineTraceByProfile(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, FName ProfileName, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)
### auto AsyncSweepByChannel(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, const FQuat& Rot, ECollisionChannel TraceChannel, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam, const FCollisionResponseParams& ResponseParam = FCollisionResponseParams::DefaultResponseParam)
### auto AsyncSweepByObjectType(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, const FQuat& Rot, const FCollisionObjectQueryParams& ObjectQueryParams, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)
### auto AsyncSweepByProfile(const UObject* WorldContextObject, EAsyncTraceType InTraceType, const FVector& Start, const FVector& End, const FQuat& Rot, FName ProfileName, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)
### auto AsyncOverlapByChannel(const UObject* WorldContextObject, const FVector& Pos, const FQuat& Rot, ECollisionChannel TraceChannel, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam, const FCollisionResponseParams& ResponseParam = FCollisionResponseParams::DefaultResponseParam)
### auto AsyncOverlapByObjectType(const UObject* WorldContextObject, const FVector& Pos, const FQuat& Rot, const FCollisionObjectQueryParams& ObjectQueryParams, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)
### auto AsyncOverlapByProfile(const UObject* WorldContextObject, const FVector& Pos, const FQuat& Rot, FName ProfileName, const FCollisionShape& CollisionShape, const FCollisionQueryParams& Params = FCollisionQueryParams::DefaultQueryParam)

关于参数和查询本身的更多信息，请参见 UWorld 上的同名函数。

这些函数会发起一次异步碰撞查询，并返回一个对象；
await 它，协程会在查询完成时恢复。
返回值可以复制，副本指向同一次查询。

await 这些函数所返回对象的右值可能效率更高，因为结果会是一个 TArray 右值，
而不是一个不可移动的 const 引用。
await 右值会使该 awaiter 的其他所有副本失效（如果有副本的话）。

可以在 await 第一个查询之前发起多个异步查询让它们并行，以获得更高的吞吐量。
await 一个已经完成的查询会同步继续执行。

await 表达式的结果取决于函数名，以及返回值是作为左值还是右值被 await 的：

|    |函数名含 `Trace` 或 `Sweep`  |函数名含 `Overlap`             |
|----|-----------------------------|-------------------------------|
|左值|`const TArray<FHitResult>&`  |`const TArray<FOverlapResult>&`|
|右值|`TArray<FHitResult>`         |`TArray<FOverlapResult>`       |

示例：
```cpp
using namespace UE5Coro::Latent;

// 最简单的用法毫不费力地避免了所有拷贝：
TArray<FHitResult> Result1 = co_await AsyncLineTraceByObjectType(this, /*...*/);

// 三个并行的查询
auto Query2 = AsyncLineTraceByChannel(this, /*...*/);
auto Query3A = AsyncSweepByProfile(this, /*...*/);
auto Query3B = Query3;
TArray<FOverlapResult> Result4 = co_await AsyncOverlapByChannel(this, /*...*/); // 移动

TArray<FHitResult> Result2A = co_await Query2; // 拷贝
const TArray<FHitResult>& Result2B = co_await Query2; // 指向 Query2 内部的引用
TArray<FHitResult> Result3 = co_await std::move(Query3A); // 从 Query3A 移出
// Query3A 和 Query3B 现在都已失效
```
