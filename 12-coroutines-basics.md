# Coroutines 协程

## Coroutines概念
Coroutines(协程), 计算机程序组件, 通过允许任务挂起和恢复执行, 来支持非抢占式的多任务. (见[Wiki](https://en.wikipedia.org/wiki/Coroutine)).

协程主要是为了异步, 非阻塞的代码. 这个概念并不是Kotlin特有的, Go, Python等多个语言中都有支持.

## Kotlin Coroutines
Kotlin中用协程来做异步和非阻塞任务, 主要优点是代码可读性好, 不用回调函数. (用协程写的异步代码乍一看很像同步代码.)

Kotlin对协程的支持是在语言级别的, 在标准库中只提供了最低程度的APIs, 然后把很多功能都代理到库中.

Kotlin中只加了`suspend`作为关键字.
`async`和`await`不是Kotlin的关键字, 也不是标准库的一部分.

比起futures和promises, kotlin中`suspending function`的概念为异步操作提供了一种更安全和不易出错的抽象.

`kotlinx.coroutines`是协程的库, 为了使用它的核心功能, 项目需要增加`kotlinx-coroutines-core`的依赖.

## Coroutines Basics: 协程到底是什么?
先上一段官方的demo:
```
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch


fun main() {
    GlobalScope.launch { // launch a new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello,") // main thread continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
```
这段代码的输出: 
先打印Hello, 延迟1s之后, 打印World.

对这段代码的解释: 

`launch`开始了一个计算, 这个计算是可挂起的(suspendable), 它在计算过程中, 释放了底层的线程, 当协程执行完成, 就会恢复(resume). 

这种可挂起的计算就叫做一个协程(coroutine). 所以我们可以简单地说`launch`开始了一个新的协程.

注意, 主线程需要等待协程结束, 如果注释掉最后一行的`Thread.sleep(2000L)`, 则只打印Hello, 没有World.

### 协程和线程的关系
coroutine(协程)可以理解为轻量级的线程. 多个协程可以并行运行, 互相等待, 互相通信. 协程和线程的最大区别就是协程非常轻量(cheap), 我们可以创建成千上万个协程而不必考虑性能.

协程是运行在线程上可以被挂起的运算. 可以被挂起, 意味着运算可以被暂停, 从线程移除, 存储在内存里. 此时, 线程就可以自由做其他事情. 当计算准备好继续进行时, 它会返回线程(但不一定要是同一个线程).

默认情况下, 协程运行在一个共享的线程池里, 线程还是存在的, 只是一个线程可以运行多个协程, 所以线程没必要太多.

### 调试
在上面的代码中加上线程的名字:
```
fun main() {
    GlobalScope.launch {
        // launch a new coroutine in background and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World! + ${Thread.currentThread().name}") // print after delay
    }
    println("Hello, + ${Thread.currentThread().name}") // main thread continues while coroutine is delayed
    Thread.sleep(2000L) // block main thread for 2 seconds to keep JVM alive
}
```
可以在IDE的Edit Configurations中设置VM options: `-Dkotlinx.coroutines.debug`, 运行程序, 会在log中打印出代码运行的协程信息:
```
Hello, + main
World! + DefaultDispatcher-worker-1 @coroutine#1
```

### suspend function
上面例子中的`delay`方法是一个`suspend function`.
`delay()`和`Thread.sleep()`的区别是: `delay()`方法可以在不阻塞线程的情况下延迟协程. (It doesn't block a thread, but only suspends the coroutine itself). 而`Thread.sleep()`则阻塞了当前线程.

所以, suspend的意思就是协程作用域被挂起了, 但是当前线程中协程作用域之外的代码不被阻塞.

如果把`GlobalScope.launch`替换为`thread`, delay方法下面会出现红线报错:
```
Suspend functions are only allowed to be called from a coroutine or another suspend function
```
suspend方法只能在协程或者另一个suspend方法中被调用.


在协程等待的过程中, 线程会返回线程池, 当协程等待结束, 协程会在线程池中一个空闲的线程上恢复. (The thread is returned to the pool while the coroutine is waiting, and when the waiting is done, the coroutine resumes on a free thread in the pool.)


## 启动协程
启动一个新的协程, 常用的主要有以下几种方式: 
* `launch`
* `async` 
* `runBlocking`

它们被称为`coroutine builders`. 不同的库可以定义其他更多的构建方式.

### runBlocking: 连接blocking和non-blocking的世界
`runBlocking`用来连接阻塞和非阻塞的世界. 

`runBlocking`可以建立一个阻塞当前线程的协程. 所以它主要被用来在main函数中或者测试中使用, 作为连接函数.

比如前面的例子可以改写成:
```
fun main() = runBlocking<Unit> {
    // start main coroutine
    GlobalScope.launch {
        // launch a new coroutine in background and continue
        delay(1000L)
        println("World! + ${Thread.currentThread().name}")
    }
    println("Hello, + ${Thread.currentThread().name}") // main coroutine continues here immediately
    delay(2000L) // delaying for 2 seconds to keep JVM alive
}
```
最后不再使用`Thread.sleep()`, 使用`delay()`就可以了.
程序输出:
```
Hello, + main @coroutine#1
World! + DefaultDispatcher-worker-1 @coroutine#2
```


### launch: 返回Job
上面的例子delay了一段时间来等待一个协程结束, 不是一个好的方法. 

`launch`返回`Job`, 代表一个协程, 我们可以用`Job`的`join()`方法来显式地等待这个协程结束:
```
fun main() = runBlocking {
    val job = GlobalScope.launch {
        // launch a new coroutine and keep a reference to its Job
        delay(1000L)
        println("World! + ${Thread.currentThread().name}")
    }
    println("Hello, + ${Thread.currentThread().name}")
    job.join() // wait until child coroutine completes
}
```
输出结果和上面是一样的.

`Job`还有一个重要的用途是`cancel()`, 用于取消不再需要的协程任务.

### async: 从协程返回值
`async`开启线程, 返回`Deferred<T>`, `Deferred<T>`是`Job`的子类, 有一个`await()`函数, 可以返回协程的结果.

`await()`也是suspend函数, 只能在协程之内调用.

```
fun main() = runBlocking {
    // @coroutine#1
    println(Thread.currentThread().name)
    val deferred: Deferred<Int> = async {
        // @coroutine#2
        loadData()
    }
    println("waiting..." + Thread.currentThread().name)
    println(deferred.await()) // suspend @coroutine#1
}

suspend fun loadData(): Int {
    println("loading..." + Thread.currentThread().name)
    delay(1000L) // suspend @coroutine#2
    println("loaded!" + Thread.currentThread().name)
    return 42
}
```
运行结果:
```
main @coroutine#1
waiting...main @coroutine#1
loading...main @coroutine#2
loaded!main @coroutine#2
42
```

## Context, Dispatcher和Scope
看一下`launch`方法的声明:
```
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
...
}
```
其中有几个相关概念我们要了解一下.

协程总是在一个context下运行, 类型是接口`CoroutineContext`. 协程的context是一个索引集合, 其中包含各种元素, 重要元素就有`Job`和dispatcher. `Job`代表了这个协程, 那么dispatcher是做什么的呢?

构建协程的coroutine builder: `launch`, `async`, 都是`CoroutineScope`类型的扩展方法. 查看`CoroutineScope`接口, 其中含有`CoroutineContext`的引用. scope是什么? 有什么作用呢?

下面我们就来回答这些问题.

### Dispatchers和线程
Context中的`CoroutineDispatcher`可以指定协程运行在什么线程上. 可以是一个指定的线程, 线程池, 或者不限.

看一个例子:
```
fun main() = runBlocking<Unit> {
    launch {
        // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) {
        // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) {
        // will get dispatched to DefaultDispatcher
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) {
        // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}
```
运行后打印出:
```
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

API提供了几种选项:
* `Dispatchers.Default`代表使用JVM上的共享线程池, 其大小由CPU核数决定, 不过即便是单核也有两个线程. 通常用来做CPU密集型工作, 比如排序或复杂计算等. 
* `Dispatchers.Main`指定主线程, 用来做UI更新相关的事情. (需要添加依赖, 比如`kotlinx-coroutines-android`.) 如果我们在主线程上启动一个新的协程时, 主线程忙碌, 这个协程也会被挂起, 仅当线程有空时会被恢复执行.
* `Dispatchers.IO`: 采用on-demand创建的线程池, 用于网络或者是读写文件的工作.
* `Dispatchers.Unconfined`: 不指定特定线程, 这是一个特殊的dispatcher.

如果不明确指定dispatcher, 协程将会继承它被启动的那个scope的context(其中包含了dispatcher). 

在实践中, 更推荐使用外部scope的dispatcher, 由调用方决定上下文. 这样也方便测试.

`newSingleThreadContext`创建了一个线程来跑协程, 一个专注的线程算是一种昂贵的资源, 在实际的应用中需要被释放或者存储复用.

切换线程还可以用`withContext`, 可以在指定的协程context下运行代码, 挂起直到它结束, 返回结果. 
另一种方式是新启一个协程, 然后用`join`明确地挂起等待.

在Android这种UI应用中, 比较常见的做法是, 顶部协程用`CoroutineDispatchers.Main`, 当需要在别的线程上做一些事情的时候, 再明确指定一个不同的dispatcher.

### Scope是什么?
当`launch`, `async`或`runBlocking`开启新协程的时候, 它们自动创建相应的scope. 所有的这些方法都有一个带receiver的lambda参数, 默认的receiver类型是`CoroutineScope`.

IDE会提示`this: CoroutineScope`:
```
launch { /* this: CoroutineScope */
}
```
当我们在`runBlocking`, `launch`, 或`async`的大括号里面再创建一个新的协程的时候, 自动就在这个scope里创建:
```
fun main() = runBlocking {
    /* this: CoroutineScope */
    launch { /* ... */ }
    // the same as:
    this.launch { /* ... */ }
}
```
因为`launch`是一个扩展方法, 所以上面例子中默认的receiver是`this`.
这个例子中`launch`所启动的协程被称作外部协程(`runBlocking`启动的协程)的child. 这种"parent-child"的关系通过scope传递: child在parent的scope中启动.

**协程的父子关系:**
* 当一个协程在另一个协程的scope中被启动时, 自动继承其context, 并且新协程的Job会作为父协程Job的child.

所以, 关于scope目前有两个关键知识点:
* 我们开启一个协程的时候, 总是在一个`CoroutineScope`里.
* Scope用来管理不同协程之间的父子关系和结构. 

协程的父子关系有以下两个特性:
* 父协程被取消时, 所有的子协程都被取消.
* 父协程永远会等待所有的子协程结束.

值得注意的是, 也可以不启动协程就创建一个新的scope. 创建scope可以用工厂方法: `MainScope()`或`CoroutineScope()`.

`coroutineScope()`方法也可以创建scope. 当我们需要以结构化的方式在suspend函数内部启动新的协程, 我们创建的新的scope, 自动成为suspend函数被调用的外部scope的child.

所以上面的父子关系, 可以进一步抽象到, 没有parent协程, 由scope来管理其中所有的子协程.
(注意: 实际上scope会提供默认job, `cancel`操作是由scope中的job支持的.)

Scope在实际应用中解决什么问题呢? 如果我们的应用中, 有一个对象是有自己的生命周期的, 但是这个对象又不是协程, 比如Android应用中的Activity, 其中启动了一些协程来做异步操作, 更新数据等, 当Activity被销毁的时候需要取消所有的协程, 来避免内存泄漏. 我们就可以利用`CoroutineScope`来做这件事: 创建一个`CoroutineScope`对象和activity的生命周期绑定, 或者让activity实现`CoroutineScope`接口.

所以, scope的主要作用就是记录所有的协程, 并且可以取消它们.

```
A CoroutineScope keeps track of all your coroutines, and it can cancel all of the coroutines started in it.
```

### Structured Concurrency
这种利用scope将协程结构化组织起来的机制, 被称为"structured concurrency".
好处是:
* scope自动负责子协程, 子协程的生命和scope绑定.
* scope可以自动取消所有的子协程. 
* scope自动等待所有的子协程结束. 如果scope和一个parent协程绑定, 父协程会等待这个scope中所有的子协程完成.

通过这种结构化的并发模式: 我们可以在创建top级别的协程时, 指定主要的context一次, 所有嵌套的协程会自动继承这个context, 只在有需要的时候进行修改即可.


### GlobalScope: daemon
`GlobalScope`启动的协程都是独立的, 它们的生命只受到application的限制. 即`GlobalScope`启动的协程没有parent, 和它被启动时所在的外部的scope没有关系.

`launch(Dispatchers.Default) { ... }`和`GlobalScope.launch { ... }`用的dispatcher是一样的.

`GlobalScope`启动的协程并不会保持进程活跃. 它们就像daemon threads(守护线程)一样, 如果JVM发现没有其他一般的线程, 就会关闭.

## Key takeaways
* Coroutine协程机制: suspend, resume, 简化回调代码.
* suspend方法.
* 启动协程的几种方法.
* Dispatcher指定线程.
* Structured Concurrency: 依靠scope来架构化管理协程.


## 参考
* [Coroutine Wiki](https://en.wikipedia.org/wiki/Coroutine)
* [官方文档 Overview页](https://kotlinlang.org/docs/reference/coroutines-overview.html)
* [官方文档 Coroutines Guide](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html)
* [Asynchronous Programming Techniques](https://kotlinlang.org/docs/tutorials/coroutines/async-programming.html)
* [Your first coroutine with Kotlin](https://kotlinlang.org/docs/tutorials/coroutines/coroutines-basic-jvm.html)
* [Introduction to Coroutines and Channels](https://play.kotlinlang.org/hands-on/Introduction%20to%20Coroutines%20and%20Channels/01_Introduction)
* [Github: Kotlin/kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)
* [Github: Coroutines Guide](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md)
* [Github: KEEP: Kotlin Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#coroutines-overview)

第三方博客:
* [Coroutines on Android (part I): Getting the background](https://medium.com/androiddevelopers/coroutines-on-android-part-i-getting-the-background-3e0e54d20bb)
* [Async Operations with Kotlin Coroutines — Part 1](https://proandroiddev.com/async-operations-with-kotlin-coroutines-part-1-c51cc581ad33)
* [Kotlin Coroutines Tutorial for Android](https://www.raywenderlich.com/1423941-kotlin-coroutines-tutorial-for-android-getting-started)
* [Coroutine Context and Scope](https://medium.com/@elizarov/coroutine-context-and-scope-c8b255d59055)