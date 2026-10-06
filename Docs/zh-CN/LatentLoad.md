# 资产加载

[English](../LatentLoad.md) | 简体中文

这些函数位于 `UE5Coro::Latent` 命名空间，因此只能在游戏线程上使用。

本页的每个函数返回的类型都满足 TLatentAwaiter 概念。

### auto AsyncLoadObject\<T\>(TSoftObjectPtr\<T\>, TAsyncLoadPriority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadObjects\<T\>(const TArray<TSoftObjectPtr\<T\>>&, TAsyncLoadPriority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadObjects(TArray\<FSoftObjectPath\>, TAsyncLoadPriority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadClass(TSoftClassPtr\<\>, TAsyncLoadPriority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadClasses(const TArray\<TSoftClassPtr\<\>\>&, TAsyncLoadPriority = FStreamableManager::DefaultAsyncLoadPriority)

这些函数开始加载传入的软指针，并返回一个 awaiter，在加载完成时恢复调用方协程。

如果所有东西都已加载，调用方协程会立即继续。
可以在 await 第一个加载之前发起多个加载，以获得更高的整体吞吐量。
在被 await 之前，awaiter 会强引用已加载的对象。

await 表达式的结果取决于函数的第一个参数：

|`co_await` 结果  |加载的内容                 |
|-----------------|---------------------------|
|`T*`             |`TSoftObjectPtr<T>`        |
|`UClass*`        |`TSoftClassPtr<>`          |
|`TArray<T*>`     |`TArray<TSoftObjectPtr<T>>`|
|`TArray<UClass*>`|`TArray<TSoftClassPtr<>>`  |
|`void`           |`TArray<FSoftObjectPath>`  |

FSoftObjectPath 重载的开销比模板化的 TSoftObjectPtr 重载更低，因为它不解析已加载的对象
（也不会因额外的模板实例化而让编译出的二进制膨胀）。
要加载“任意对象”，可以用 TSoftObjectPtr\<UObject\> 代替 FSoftObjectPath。

对于带类型的 TSoftClassPtr，可以用 TSubclassOf\<T\> 接收返回的指针，
因为它可以从 UClass* 隐式转换。

示例：
```cpp
using namespace UE5Coro::Latent;

UClass* Class = co_await AsyncLoadClass(SoftClass);
if (AActor* SpawnedActor = GetWorld()->SpawnActor<AActor>(Class))
    SpawnedActor->Act();
```

### auto AsyncPreloadPrimaryAssets(const TArray\<FPrimaryAssetId\>& AssetsToLoad, const TArray\<FName\>& LoadBundles, bool bLoadRecursive, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)

本函数开始预加载由主资产 ID 指定的资产。
可以 await 它的返回值，把协程挂起到预加载完成为止。

await 表达式的结果是来自 UAssetManager 的 TSharedPtr\<FStreamableHandle\>，
**必须**把它保存下来，以防资产被卸载。

如果不想手动处理这个句柄，请参见下面更方便的
AsyncLoadPrimaryAsset\<T\> 和 AsyncLoadPrimaryAssets\<T\> 函数。

示例：
```c++
using namespace UE5Coro::Latent;

TSharedPtr<FStreamableHandle> Handle = co_await AsyncPreloadPrimaryAssets(
    Assets, Bundles, bRecursive, Priority);
```

### auto AsyncLoadPrimaryAsset(const FPrimaryAssetId& AssetToLoad, const TArray\<FName\>& LoadBundles = \{\}, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadPrimaryAsset\<T\>(FPrimaryAssetId AssetToLoad, const TArray\<FName\>& LoadBundles = \{\}, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadPrimaryAssets(TArray\<FPrimaryAssetId\> AssetsToLoad, const TArray\<FName\>& LoadBundles = \{\}, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncLoadPrimaryAssets\<T\>(TArray\<FPrimaryAssetId\> AssetsToLoad, const TArray\<FName\>& LoadBundles = \{\}, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)

这些函数使用主资产 ID，并支持指定资产包（bundle）。
调用函数时开始加载，返回值可用于在加载完成时恢复调用方协程。
可以在 await 第一个操作之前让多个操作并行，以获得更高的吞吐量。

如果资产已经加载，调用方协程会立即继续。

已加载的对象会一直留在内存中，直到被显式卸载。

await 表达式的结果是：
* 非模板版本：`void`
* AsyncLoadPrimaryAsset\<T\>：`T*`
* AsyncLoadPrimaryAssets\<T\>：`TArray<T*>`

示例：
```cpp
using namespace UE5Coro::Latent;

for (auto* Asset : co_await AsyncLoadPrimaryAssets<UExampleAsset>(...))
    DoSomethingWith(Asset);
```

### auto AsyncLoadPackage(const FPackagePath& Path, FName PackageNameToCreate = NAME_None, EPackageFlags PackageFlags = PKG_None, int32 PIEInstanceID = INDEX_NONE, TAsyncLoadPriority PackagePriority = 0, const FLinkerInstancingContext* InstancingContext = nullptr)

开始加载指定路径上的 UPackage，返回一个对象，
await 它，调用方协程会在加载完成时恢复。
如果 co_await 时包已经加载，则什么都不会发生，调用方协程立即继续。

await 表达式的结果是加载出来的 UPackage*。

参数的更多细节请参见全局命名空间中的引擎函数 `LoadPackageAsync()`。

示例：
```cpp
using namespace UE5Coro::Latent;

UPackage* Package = co_await AsyncLoadPackage(Path);
```

### auto AsyncChangeBundleStateForPrimaryAssets(const TArray<FPrimaryAssetId>& AssetsToChange, const TArray<FName>& AddBundles, const TArray<FName>& RemoveBundles, bool bRemoveAllBundles = false, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)
### auto AsyncChangeBundleStateForMatchingPrimaryAssets(const TArray<FName>& NewBundles, const TArray<FName>& OldBundles, TAsyncLoadPriority Priority = FStreamableManager::DefaultAsyncLoadPriority)

这两个函数对传入的（第一个函数）或所有匹配的（第二个函数）主资产
开始执行请求的资产包状态变更，并返回一个对象；
await 它，调用方协程会在加载完成或被取消时恢复。

如果资产管理器判断无事可做，调用方协程会立即继续。

更多细节请参见 `UAssetManager` 上的对应函数。

示例：
```cpp
using namespace UE5Coro::Latent;

co_await AsyncChangeBundleStateForMatchingPrimaryAssets({"LoadMe"}, {"RemoveMe"});
```
