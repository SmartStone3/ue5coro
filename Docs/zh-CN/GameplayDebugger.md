# Gameplay Debugger 集成

[English](../GameplayDebugger.md) | 简体中文

UE5Coro 内置了对 Unreal Engine 自带 Gameplay Debugger 的支持，
可以全局显示当前正在运行的协程，或显示选中 actor 上的协程。
协程可以
[配合](Coroutine.md#static-void-tcoroutinesetdebugnamefstring-name)提供
自身最新的调试描述，便于查看。

另外附赠了一个本地化功能，它可以脱离调试器独立使用。

<a id="setup"></a>
## 设置

调试器要求 UE5Coro 在构建时开启协程追踪；由于它会影响全局性能，默认是关闭的。
要开启它，请在当前使用的 Target.cs 中加入下面这一行：

```cs
UE5CoroModuleRules.bEnableGameplayDebuggerIntegration = true;
```

这会激活 UE5Coro 的 Gameplay Debugger 类别，无需其他步骤。

<sup>
是否开启了协程追踪由 UE5CORO_ENABLE_COROUTINE_TRACKING 宏指示，
可以用它来包裹只应在这种特定情况下运行的代码。
如果目的只是把代码排除在 Shipping 构建之外，应优先使用 UE5CORO_DEBUG
或 Unreal 内置的 UE_BUILD_* 宏。
</sup>

<a id="optional-enable-conditional-globally"></a>
### 可选：全局启用 `conditional`

如果你想在项目的本地化中使用 `conditional` 文本格式化修饰符，
请在**每一个** Target.cs 中加入这一行：

```cs
UE5CoroDebug.bRegisterConditionalTextFormatArgumentModifier = true;
```

否则 UE5Coro 会把它留给自己用，并在 Shipping 构建中不干扰文本格式化。

## 使用

启用 Gameplay Debugger 和 UE5Coro 类别之后（具体做法请参考 Unreal Engine 自己的
Gameplay Debugger 文档），会显示整个引擎中正在运行的、由 UE5Coro 管理的协程列表：

![UE5Coro Gameplay Debugger 截图](../GameplayDebugger.avif)

> [!TIP]
> 非英文编辑器完全支持本地化。
> 这些文本位于 UE5Coro 命名空间中，可以在本地化面板（Localization Dashboard）中
> 从文本文件收集。它们使用内部的 `ue5coro_conditional` 修饰符，
> 该修饰符只在启用 Gameplay Debugger 菜单时可用。

如果选中了某个 actor 进行调试，它的协程会单独显示。
**只有 latent 协程能与 actor 关联。**

屏幕上显示的协程数量由 CVar `UE5Coro.MaxDisplayedCoroutines` 和
`UE5Coro.MaxDisplayedCoroutinesOnTarget` 控制。

每一条目中，协程模式旁边的数字是它的 DebugID[^noid]，
支持 natvis 的调试器也会显示这个数字。

[^noid]: 在非 DEBUG 的 UE5Coro 下启用 Gameplay Debugger 支持并不常见，但也是可以的。
         这种情况下不会生成调试 ID，所有协程都会显示为 #-1。

带引号的描述（截图上的函数名）是一个由协程控制的任意 FString，示例见
[TCoroutine<>::SetDebugName()](Coroutine.md#static-void-tcoroutinesetdebugnamefstring-name)。

`[Ticking]` 表示一个异步协程位于游戏线程上，正借助一个临时的 latent 动作，
以便能够 `co_await` 一个 latent awaiter。
更多细节见[异步模式](Coroutine.md#async-mode)。

`[Detached]` 表示一个 latent 协程被暂时固定（pinned）并与游戏线程分离，以防被过早销毁。
当它 await 一个非 latent 的 awaiter 时会出现这种情况，通常是为了在协程运行于
其他线程时防止意外的垃圾回收，或者确保回调有一个可以返回的对象。
更多细节见 [Latent 模式](Coroutine.md#latent-mode)。

<a id="the-conditional-modifier"></a>
## `conditional` 修饰符

[启用](#optional-enable-conditional-globally)之后，`conditional` 会根据其参数，
在输出中排除或插入一段格式化的子串。
如果条件不受当地文化或语法规则的影响，可以用它来代替 `gender` 或 `plural`。
支持嵌套的格式字符串，但 Unreal Engine 本身对嵌套括号的解析是错误的。

下列值不会产生任何输出：`false`、`0`、`0U`、`±0.0f`、`±0.0`、`""`、
所有性别（gender），以及其他任何（按文化感知方式）格式化后为空字符串的值。
其他任何值都会把格式化后的条件文本插入到输出中。

示例：
```c++
FText Format = LOCTEXT("TitleExample",
    "{Name}'s Adventures{Location}|conditional( in {Location})");

// Alice's Adventures
FText::FormatNamed(Format,
    TEXT("Name"), LOCTEXT("Protagonist", "Alice"),
    TEXT("Location"), INVTEXT(""));

// Alice's Adventures in Wonderland
FText::FormatNamed(Format,
    TEXT("Name"), LOCTEXT("Protagonist", "Alice"),
    TEXT("Location"), LOCTEXT("Setting", "Wonderland"));
```

```c++
FText Format = LOCTEXT("QuoteExample", "{0}|conditional(\"{0}\")");

// "Text"，包括引号
FText::FormatOrdered(Format, INVTEXT("Text"));

// 一个空文本，而不是 ""
FText::FormatOrdered(Format, INVTEXT(""));
```

> [!IMPORTANT]
> Unreal 不会对没有对应参数的修饰符求值。
> ```c++
> // "123{1}|conditional(hidden?)"
> FText::FormatOrdered(INVTEXT("{0}{1}|conditional(hidden?)"), 123);
>
> // "123"
> FText::FormatOrdered(INVTEXT("{0}{1}|conditional(hidden?)"), 123, false);
> ```
