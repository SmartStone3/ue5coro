# HTTP

[English](../Http.md) | 简体中文

`UE5Coro::Http` 命名空间目前只有一个函数，用于异步执行 HTTP 请求。

HTTP API 的大部分内容仍在引擎的 IHttpRequest 上。

### auto ProcessAsync(FHttpRequestRef Request)

本函数会对传入的参数调用 ProcessRequest()，并返回一个对象；
await 它，协程会在请求完成时恢复。

await 表达式的结果是一个由 FHttpResponsePtr 和 bConnectedSuccessfully 组成的 TTuple。

示例：
```cpp
using namespace UE5Coro::Http;

FHttpRequestRef Request = FHttpModule::Get().CreateRequest();
Request->SetURL(TEXT("https://www.example.com"));
if (auto [Response, bConnectedSuccessfully] = co_await ProcessAsync(Request);
    Response && bConnectedSuccessfully)
{
    FString Content = Response->GetContentAsString();
    // ...
}
```

> [!WARNING]
> Unreal Engine 5.3 引入了 EHttpRequestDelegateThreadPolicy。
> 它受到完全支持，不过它在引擎中首次发布时是有问题的。
>
> 使用 EHttpRequestDelegateThreadPolicy::CompleteOnHttpThread 可能导致请求永远卡住，
> 进而导致协程无法恢复。
