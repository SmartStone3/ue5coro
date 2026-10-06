# Latent 链

[English](../LatentChain.md) | 简体中文

这些函数用于在 C++ 中调用那些不认识协程的 latent 函数，并让调用方 await 它们的执行，
用法类似于在蓝图中把 🕒 节点串联起来。

它们的返回类型满足 TLatentAwaiter 概念，因此适用 [latent](Latent.md#latent-awaiters) 规则。
此外，这些函数要求被调用时 GWorld 有效，因为要用它来注册被调用函数所创建的 latent 动作。
在绑定游戏线程的 gameplay 代码执行期间，GWorld 通常是有效的。

只支持 latent 动作。
通过其他方式（例如 UBlueprintAsyncActionBase）创建的 🕒 节点不受支持。
要 await 这类节点的委托，请参见[委托支持](Implicit.md#delegates)。

返回的 awaiter 假定被链接的 latent 动作是世界敏感的，详见[此页](Latent.md)。

### auto Chain(? (*Function)(?...), T&&... Args)
### auto Chain(U* Object, ? (U::*Function)(?...), T&&... Args)

完整的、<!--更加-->晦涩难懂的声明在文档中略去。
第一个重载用于静态 UFUNCTION，第二个用于非静态成员 UFUNCTION。

调用这些函数与绑定委托类似，但提供实参时**必须跳过** world context 和
FLatentActionInfo 参数。
它们会被自动插入。

这些函数返回一个对象，可以在游戏线程上 co_await 它，
在 latent 动作结束时恢复调用方协程。

如果 latent 动作成功结束，即它本会从输出执行引脚继续执行调用它的蓝图，
await 表达式的结果为 `true`；如果失败，即它已结束但本不会恢复蓝图，结果为 `false`。

对于带有自定义 K2Node 的 latent 动作，K2Node 的行为**不会**被遵守，
只会遵守从 C++ 调用该函数时创建的那个核心 latent 动作。

调用示例：
```cpp
using namespace UE5Coro::Latent;

// 真要做延时，应改用 UE5Coro::Latent::Seconds，它是为 C++ 设计的，
// await 起来效率更高。
// 这个引擎函数是静态的：
// static void Delay(const UObject* WorldContextObject, float Duration,
//                   FLatentActionInfo LatentInfo)
bool bSuccess = co_await Chain(&UKismetSystemLibrary::Delay, 1.0f);

// 这个引擎函数不是静态的：
// void OpenSourceLatent(const UObject* WorldContextObject,
//     FLatentActionInfo LatentInfo, UMediaSource* MediaSource,
//     const FMediaPlayerOptions& Options, bool& bSuccess)
co_await Chain(MyMediaPlayer, &UMediaPlayer::OpenSourceLatent, MyMediaSource,
               MyOptions, bSuccess);
```

WorldContextObject 和 LatentInfo 的识别基于编译期的启发式规则：
* 第一个 UObject\* 或 UWorld\* 参数（包括指向 const 的指针）被视为 world context 对象，
  会接收 GWorld。
* `this` 不参与自动匹配，它要显式传给 Chain 的成员函数指针重载。
* 第一个 FLatentActionInfo 参数（包括 const FLatentActionInfo&）被视为 _那个_
  latent info 参数，会接收一个生成的合适值。

其余参数取自 Chain 调用，并原样转发。
左值“输出”引用会被遵守：被链接的调用会写回原始变量，
这意味着该引用必须在被链接的 latent 动作的整个执行期间保持有效。
如果它是协程的局部变量，或者是协程所属对象的成员，通常就满足这一点。

对右值引用的写入（与 UFUNCTION 无关）**不会**传播到 Chain 之外；
由于这会引入隐蔽的生命周期 bug，建议不要链接接受右值引用的函数。
引擎中大多数 latent UFUNCTION 只接受值和/或左值引用。

这些启发式规则覆盖了引擎中绝大多数 latent UFUNCTION，但并不完美。
如果某个函数的样子与预期不同，请使用下面这个函数：

### auto ChainEx(? Function, T&&... Args)

本函数的行为与 Chain 完全相同，但没有自动参数匹配，
是应对 Chain 覆盖不到的特殊函数的最后手段。

请显式提供 world context，或者用 std::placeholders::_1 代替它，
并在 latent info 参数的位置传入 std::placeholders::_2。
只有 _2 是必需的。

参数的传递方式与 std::bind 相同，值得注意的是：成员函数指针所用的对象要放在函数指针之后，
并且引用需要特殊处理。
传递左值引用时，需要用 std::ref() 或 std::cref() 包装，
否则函数引用的会是一份（可能发生对象切片的）拷贝。
传递右值引用基本上不受支持。
更多细节请参见 std::bind() 本身。

返回值与 Chain 的相同，co_await 它会得到同样表示成功与否的 bool。

用 ChainEx 改写上面的 Chain 示例：
```cpp
using namespace std::placeholders;
using namespace UE5Coro::Latent;

// 代替 Chain(&UKismetSystemLibrary::Delay, 1.0f)
bool bSuccess = co_await ChainEx(&UKismetSystemLibrary::Delay, _1, 1.0f, _2);
// 或者 ChainEx(&UKismetSystemLibrary::Delay, MyWorldContext, 1.0f, _2)

// 代替 Chain(MyMediaPlayer, &UMediaPlayer::OpenSourceLatent,
//            MyMediaSource, MyOptions, bSuccess)
co_await ChainEx(&UMediaPlayer::OpenSourceLatent, MyMediaPlayer, _1, _2,
                 MyMediaSource, std::cref(MyOptions), std::ref(bSuccess));
// 或者 ChainEx(&UMediaPlayer::OpenSourceLatent, MyMediaPlayer, this, _2,
//              MyMediaSource, std::cref(MyOptions), std::ref(bSuccess))
```
