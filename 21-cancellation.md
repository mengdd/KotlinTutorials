# 协程的取消
本文讨论协程的取消, 以及实现时可能会碰到的几个问题.

## 协程的取消
取消的意义: 避免资源浪费, 以及多余操作带来的问题.

基本特性:
- cancel scope的时候会cancel其中的所有child coroutines.
- 一旦取消一个scope, 你将不能再在其中launch新的coroutine.
- 一个在取消状态的coroutine是不能suspend的.


## Android开发中的取消
在Android开发中, 比较常见的情形是由于View生命周期的终止, 我们需要取消一些操作.

通常我们不需要手动调用`cancel()`方法, 那是因为我们利用了一些更高级的包装方法, 比如:
- `viewModelScope`: 会在ViewModel onClear的时候cancel.
- `lifecycleScope`: 会在作为Lifecycle Owner的View对象: Activity, Fragment到达DESTROYED状态时cancel.


## 取消并不是自动获得的
all suspend functions from `kotlinx.coroutines` are cancellable, but not yours.

kotlin官方提供的suspend方法都会有cancel的处理, 但是我们自己写的suspend方法就需要自己留意.
尤其是耗时或者带循环的地方, 通常需要自己加入检查, 否则即便调用了cancel, 代码也继续在执行.

有这么几种方法:
- `isActive()`
- `ensureActive()`
- `yield()`: 除了ensureActive以外, 会出让资源, 比如其他工作不需要再往线程池里加线程.


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

### 资源清理工作
finally通常用于try block之后的的资源清理, 如果其中没有suspend方法那么没有问题.

如果finally中的代码是suspend的, 如前所述, 一个在取消状态的coroutine是不能suspend的.
那么需要用一个`withContext(NonCancellable)`.

例子:
```kotlin
fun main() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
}
```

注意这个方法一般用于会suspend的资源清理, 不建议在各个场合到处使用, 因为它破坏了对coroutine执行取消的控制.

### 需要更长生命周期的工作
如果有一些工作需要比View/ViewModel更长的生命周期, 可以把它放在更下层, 用一个生命周期更长的scope. 
可以根据不同的场景设计, 比如可以用一个application生命周期的scope:

```kotlin
class MyApplication : Application() {
  // No need to cancel this scope as it'll be torn down with the process
  val applicationScope = CoroutineScope(SupervisorJob() + otherConfig)
}
```
再把这个scope注入到repository中去.

## 总结: 再看Structured concurrency


## References & Further Reading
Kotlin官方文档的网页版和markdown版本:
- [Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)
- [Cancelling and timeouts github md version](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/topics/cancellation-and-timeouts.md)

Android官方文档上链接的博客和视频:
- [Cancellation in coroutines](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629)
- [KotlinConf 2019: Coroutines! Gotta catch 'em all! by Florina Muntenescu & Manuel Vivo](https://www.youtube.com/watch?v=w0kfnydnFWI)

其他:
- [Coroutines: first things first](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)
- [Kotlin Coroutines and Flow - Use Cases on Android](https://github.com/LukasLechnerDev/Kotlin-Coroutines-and-Flow-UseCases-on-Android)
- [Structured Concurrency Anniversary](https://elizarov.medium.com/structured-concurrency-anniversary-f2cc748b2401)
- [Exceptions in coroutines](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
- [Coroutines & Patterns for work that shouldn’t be cancelled](https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad)

