# 线程原语

[English](../Threading.md) | 简体中文

这些类是对 Unreal 内置版本的替代，并支持协程。

<a id="fawaitableevent"></a>
## FAwaitableEvent

这个类的行为与 Unreal Engine 的 FEvent/FEventRef 类似，但它没有 Wait() 函数；
取而代之的是，它可以被直接 await。
所有操作都是线程安全的。

FAwaitableEvent 对象不可移动。
如果需要移动/复制，请使用智能指针。

FAwaitableEvent 支持加速取消。
取消会在与调用 Cancel() 的那个线程同类的命名线程上处理。

### FAwaitableEvent::FAwaitableEvent(EEventMode Mode = EEventMode::AutoReset, bool bInitialState = false)

以指定的模式和初始状态初始化一个新事件。
默认是一个未触发的自动重置事件，与 FEvent 和 FEventRef 一致。

### void FAwaitableEvent::Trigger()

触发事件。
如果有符合条件的协程正在 await 这个事件，它们会直接在这次调用中、
在调用方的线程上被恢复。

EEventMode::AutoReset 事件在此之后会自行清除，并且保证每次 Trigger 只恢复一个协程。

EEventMode::ManualReset 事件会放行当前所有正在 await 的协程，
即使事件在 Trigger 被调用之后、返回之前被重置了。
这种情况下，在 Reset() 调用之后才 await 这个事件的协程会被挂起。

### void FAwaitableEvent::Reset()

重置事件，之后的 await 会挂起各自的协程，直到事件下一次被触发。

### bool FAwaitableEvent::IsManualReset() const noexcept

如果该事件是以 EEventMode::ManualReset 创建的，返回 true。

## FAwaitableSemaphore

这个类的行为与 std::counting_semaphore 类似，但它没有 acquire() 方法；
取而代之的是，它可以被直接 await，await 一次获取 1 个计数。
所有操作都是线程安全的。

Unreal Engine 有好几种差异很大的 FSemaphore 类型，最常见的是 Windows 上
一个相当重量级的、有名字的跨进程信号量。
不支持跨进程使用。

FAwaitableSemaphore 对象不可移动。
如果需要移动/复制，请使用智能指针。

FAwaitableSemaphore 支持加速取消。
取消会在与调用 Cancel() 的那个线程同类的命名线程上处理。

### FAwaitableSemaphore::FAwaitableSemaphore(int Capacity = 1, int InitialCount = 1)

以指定的容量和初始计数初始化一个新信号量。
默认是一个未锁定的二元信号量。

Capacity 必须为正数，InitialCount 不能为负数，也不能大于 Capacity。

### void FAwaitableSemaphore::Unlock(int InCount = 1)

将该信号量解锁（释放）指定的次数，默认为一次。
这最多会恢复 InCount 个正在 await 该信号量的协程。

把信号量解锁到超过其容量会导致未定义行为。
