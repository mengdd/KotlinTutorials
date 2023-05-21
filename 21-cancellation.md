# 协程的取消
本文讨论协程的取消, 以及实现时可能会碰到的几个问题.

## 协程的取消
取消的意义: 避免资源浪费, 以及多余操作带来的问题.


## Android开发中的取消
在Android开发中, 比较常见的情形是由于View生命周期的终止, 我们需要取消一些操作.
通常我们不需要手动调用`cancel()`方法, 那是因为我们利用了一些更高级的包装方法.



## 取消并不是自动获得的
suspend function are cancellable, but not yours.

kotlin官方提供的suspend方法都会有cancel的处理, 但是我们自己写的suspend方法就需要自己留意.
尤其是耗时或者带循环的地方, 通常需要自己加入检查, 否则即便调用了cancel, 代码也继续在执行.

有这么几种方法:
* `isActive()`
* `ensureActive()`
* `yield()`



## catch Exception和runCatching
众所周知catch一个很general的`Exception`类型可能不是一个好做法.
因为你以为捕获了A, B, C异常, 结果实际上还有D, E, F.

在开发阶段的快速失败会帮助我们更早定位和解决问题.


协程还推出了一个"方便"的`runCatching`方法, catch`Throwable`.
让我们写出了看似更"保险", 但却更容易忽略取消机制的代码.

###
CancellationException的特殊处理

## 不想取消的处理
可能还有一些工作我们不想随着job的取消而完全取消.
比如finally中的资源清理工作.

一种方法是用`withContext(NonCancellable)`, 还可以用一个生命周期更长的scope.


## References
* [Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)
* [Cancelling and timeouts github md version](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/topics/cancellation-and-timeouts.md)
* [Kotlin Coroutines and Flow - Use Cases on Android](https://github.com/LukasLechnerDev/Kotlin-Coroutines-and-Flow-UseCases-on-Android)
* [Structured Concurrency Anniversary](https://elizarov.medium.com/structured-concurrency-anniversary-f2cc748b2401)
* [Exceptions in coroutines](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
* [Coroutines & Patterns for work that shouldn’t be cancelled](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)
