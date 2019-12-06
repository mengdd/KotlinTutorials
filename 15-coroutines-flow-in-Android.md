# Coroutines in Android - One Shot and Multiple Values
在Android中, 我们用到的数据有可能是一次性的, 也有可能是需要多个值的.

本文介绍Android中结合协程(coroutines)的MVVM模式如何处理这两种情况. 重点介绍协程`Flow`在Android中的应用. 

## One-shot vs multiple values
实际应用中要用到的数据可能是一次性获取的(one-shot), 也可能是多个值(multiple values), 或者称为流(stream).

举例, 一个微博应用中:
* 微博信息: 请求的时候获取, 结果返回即完成. -> one-shot.
* 阅读和点赞数: 需要观察持续变化的数据源, 第一次结果返回并不代表完成. -> multiple values, stream.


## MVVM构架中的数据类型
一次性操作和观察多个值(流)的数据, 在架构上看起来会有什么不同呢?
* One-shot operation: ViewModel中是`LiveData`, Repository和Data source中是`suspend fun`.
```
class MyViewModel {
    val result = liveData {
        emit(repository.fetchData())
    }
}
```

多个值的实现有两种选择:
* Multiple values with LiveData: ViewModel, Repository, Data source都返回`LiveData`. 但是`LiveData`其实并不是为流式而设计的, 所以用起来会有点奇怪.
* Streams with Flow: ViewModel中是`LiveData`, Repository和Data source返回`Flow`.

可以看出两种方式的主要不同点就是ViewModel消费的数据形式, 是`LiveData`还是`Flow`.

后面会从ViewModel, Repository和Data source三个层面来说明.

## Flow是什么
既然提到了`Flow`, 我们先来简单讲一下它是什么, 这样大家能在same page.

Kotlin中的多个值, 可以存储在集合中, 比如list, 也可以靠计算生成sequence, 但如果值是异步生成的, 需要将方法标记为`suspend`来避免阻塞主线程.

flow和sequence类似, 但flow是非阻塞的.

看这个例子:
```
fun foo(): Flow<Int> = flow {
    // flow builder
    for (i in 1..3) {
        delay(1000) // pretend we are doing something useful here
        emit(i) // emit next value
    }
}

fun main() = runBlocking<Unit> {
    // Launch a concurrent coroutine to check if the main thread is blocked
    launch {
        for (k in 1..3) {
            println("I'm not blocked $k")
            delay(1000)
        }
    }
    // Collect the flow
    foo().collect { value -> println(value) }
}
```
这段代码执行后输出:
```
I'm not blocked 1
1
I'm not blocked 2
2
I'm not blocked 3
3
```

* 这里用来构建Flow的`flow`方法是一个builder function, 在builder block里的代码可以被`suspend`.
* `emit`方法负责发送值.
* cold stream: 只有调用了terminal operation才会被激活. 最常用的是`collect()`.


如果熟悉Reactive Streams, 或用过RxJava就可以感觉到, Flow的设计看起来很类似. 


## ViewModel层 
发送单个值的情况比较简单和典型, 这里不再多说, 主要说发送多个值的情况. 每次又分ViewModel消费的类型是`LiveData`还是`Flow`两种情况来讨论.

### 发射N个值 
#### LiveData -> LiveData
```
val currentWeather: LiveData<String> = dataSource.fetchWeather()
```
#### Flow -> LiveData
```
val currentWeatherFlow: LiveData<String> = liveData {
    dataSource.fetchWeatherFlow().collect {
        emit(it)
    }
}
```

为了减少boilerplate代码, 简化写法:
```
val currentWeatherFlow: LiveData<String> = dataSource.fetchWeatherFlow().asLiveData()
```
后面都直接用这种简化的形式了.

### 发射1+N个值
#### LiveData -> LiveData
```
val currentWeather: LiveData<String> = liveData {
    emit(LOADING_STRING)
    emitSource(dataSource.fetchWeather())
}
```
`emitSource()`发送的是一个`LiveData`.

#### Flow -> LiveData
用`Flow`的时候可以用上面同样的形式:
```
val currentWeatherFlow: LiveData<String> = liveData {
    emit(LOADING_STRING)
    emitSource(
        dataSource.fetchWeatherFlow().asLiveData()
    )
}
```
这样写看起来有点奇怪, 可读性不好, 所以可以利用`Flow`的API, 写成这样:
```
val currentWeatherFlow: LiveData<String> = 
dataSource.fetchWeatherFlow()
    .onStart{emit(LOADING_STRING)}
    .asLiveData()
```

### Suspend transformation
如果想在ViewModel中做一些转换.
#### LiveData -> LiveData
```
val currentWeatherLiveData: LiveData<String> = dataSource.fetchWeather().switchMap {
    liveData { emit(heavyTransformation(it)) }
    
}
```
这里不太适合用`map`来做转换, 因为是在主线程.

#### Flow -> LiveData
用`Flow`来做转换就很方便:
```
val currentWeatherFlow: LiveData<String> = dataSource.fetchWeatherFlow()
    .map{ heavyTransformation(it) }
    .asLiveData()
```

## Repository层
Repository层通常用来组装和转换数据.
`LiveData`被设计的初衷并不是做这些转换的. 
`Flow`则提供了很多有用的操作符, 所以显然是一种更好的选择:
```
val currentWeatherFlow: Flow<String> =
    dataSource.fetchWeatherFlow()
        .map { ... }
        .filter { ... }
        .dropWhile { ... }
        .combine { ... }
        .flowOn(Dispatchers.IO)
        .onCompletion { ... }
```

## Data Source层
Data Source层是网络和数据库, 通常会用到一些第三方的库.
如果用了支持协程的库, 如Retrofit和Room, 那么只需要把方法标记为suspend的, 就行了.

* Retrofit supports coroutines from 2.6.0
* Room supports coroutines from 2.1.0

### One-shot operations
对于一次性操作比较简单, 数据层的只要`suspend`方法返回值就可以了.
```
suspend fun doOneShot(param: String) : String = retrofitClient.doSomething(param)
```

如果所用的网络或者数据库不支持协程, 有办法吗? 答案是肯定的.
用`suspendCoroutine`来解决.

比如你用的第三方库是基于callback的, 可以用`suspendCancellableCoroutine`来改造one-shot operation:
```
suspend fun doOneShot(param: String): Result<String> = 
suspendCancellableCoroutine { continuation -> 
    api.addOnCompleteListener { result -> 
        continuation.resume(result)
    }.addOnFailureListener { error -> 
        continuation.resumeWithException(error)
    }.fetchSomething(param)
}
```
如果协程被取消了, 那么resume会被忽略.

验证代码如期工作后, 可以做进一步的重构, 把这部分抽象出来.

### Data source with Flow
数据层返回`Flow`, 可以用`flow` builder:
```
fun fetchWeatherFlow(): Flow<String> = flow {
    var counter = 0
    while(true) {
        counter++
        delay(2000)
        emit(weatherConditions[counter % weatherConditions.size])
    }
}

```

如果你所用的库不支持Flow, 而是用回调, `callbackFlow` builder可以用来改造流.

```
fun flowFrom(api: CallbackBasedApi): Flow<T> = callbackFlow {
    val callback = object: Callback {
        override fun onNextValue(value: T) {
            offer(value)
        }
        
        override fun onApiError(cause: Throwable) {
            close(cause)
        }
        
        override fun onCompleted() = close()
    }
    api.register(callback)
    awaitClose { api.unregister(callback) }
}
```

## 可能并不需要LiveData
在上面的例子中, ViewModel仍然保持了自己向UI暴露的数据是`LiveData`类型. 那么有没有可能不用`LiveData`呢?

```
lifecycleScope.launchWhenStarted {
    viewModel.flowToFlow.collect {
        binding.currentWeather.text = it
    }
}
```
这样其实和用`LiveData`是一样的效果.

## 参考
视频:
* [LiveData with Coroutines and Flow (Android Dev Summit '19)](https://www.youtube.com/watch?v=B8ppnjGPAGE&list=PLWz5rJ2EKKc_xXXubDti2eRnIKU0p7wHd&index=4)

文档:
* [Kotlin官方文档: Flow](https://kotlinlang.org/docs/reference/coroutines/flow.html)

博客:
* [Coroutines On Android (part III): Real work](https://medium.com/androiddevelopers/coroutines-on-android-part-iii-real-work-2ba8a2ec2f45)
* [Lessons learnt using Coroutines Flow in the Android Dev Summit 2019 app](https://medium.com/androiddevelopers/lessons-learnt-using-coroutines-flow-4a6b285c0d06)