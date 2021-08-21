# Kotlin Flows

本文想要回答的问题:
* Flow是什么, 怎么用?
* Flow的不同类型, StateFlow, SharedFlow区别?
* Flow使用时的注意事项?
* MVVM pattern中, Flow的角色和位置?
* 操作符`stateIn`, `shareIn`的用法和区别?
* StateFlow和LiveData的比较?
* 在Compose应用中, Flow和State的使用? 各自的作用?
* Flow的单元测试

## Coroutines Flow Basics
### Flow是什么
Flow可以按顺序发送多个值, 概念上是一个数据流, 发射的值必须是同一个类型.
Flow使用suspend方法来生产/消费值, 数据流可以做异步计算.

几个基本知识点:
* 创建flow: 通过[flow builders](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)
* Flow数据流通过`emit()`来发射元素.
* 可以通过各种操作符对flow的数据进行处理. 注意中间的操作符都不会触发flow的数据发送.
* Flow默认是cold flow, 即需要通过被观察才能激活, 最常用的操作符是`collect()`.
* Flow的`CoroutineContext`, 不指定的情况下是`collect()`的`CoroutineContext`, 如果想要更改, 用[flowOn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html)
改之前的.

关于Flow的基本用法, 19年底写的这篇[coroutines flow in Android](./15-coroutines-flow-in-Android.md)可以温故知新.

### Flow的类型
StateFlow继承于SharedFlow, SharedFlow继承于Flow.

* [Flow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)
基类. Cold.
Flow的两大特性: Context preservation; Exception transparency.

* [SharedFlow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/index.html)
继承Flow, 是一种hot flow, 所有collectors共享它的值, 永不终止, 是一种广播的方式. 
一个shared flow上的活跃collector被叫作subscriber.

在sharedFlow上的collect call永远不会正常complete, 还有Flow.launchIn.
可以配置replay and buffer overflow strategy.

* [StateFlow](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/index.html)
继承SharedFlow, hot flow, 和是否有collector收集无关, 永不complete.

可以通过`value`属性访问当前值.
有conflated特性, 会跳过太快的更新, 永远返回最新值.
Strong equality-based conflation: 会通过`equals()`来判断值是否发生改变, 如果没有改变, 则不会通知collector.

#### cold vs hot
cold stream 可以重复收集, 每次收集, 会对每一个收集者单独开启一次.
hot stream 永远发射不同的值, 和是否有人收集无关, 永远不会终止.

* [sharedIn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/share-in.html)
可以把cold flow转成hot的SharedFlow.
* [stateIn](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html)
可以把cold flow转成hot的StateFlow.

#### StateFlow vs SharedFlow
共性:
* `StateFlow`和`SharedFlow`永远都不会停止. 不能指望它们的`onCompletionCallback`.

不同点:
* `StateFlow`可以通过`value`属性读到最新的值, 但`SharedFlow`却不行.
* `StateFlow`是conflated: 如果新的值和旧的值一样, 不会传播.
* `SharedFlow`需要合理设置buffer和replay策略.


互相转换:
SharedFlow用了`distinctUntilChanged`以后变成StateFlow.

```kotlin
// MutableStateFlow(initialValue) is a shared flow with the following parameters:
val shared = MutableSharedFlow(
    replay = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
shared.tryEmit(initialValue) // emit the initial value
val state = shared.distinctUntilChanged() // get StateFlow-like behavior
```

### Flow的操作符
一个Flow操作符的可视化小网站: [FlowMarbles](https://flowmarbles.com/).

## Use Flow in Android
### StateFlow
`StateFlow`是一个state-holder, 可以通过`value`读到当前状态值.
一般会有一个`MutableStateFlow`类型的Backing property.

`StateFlow`是hot的, collect并不会触发producer code.
当有新的consumer时, 新的consumer会接到上次的状态和后续的状态.

### StateFlow vs LiveData
`StateFlow`和`LiveData`很像.

`StateFlow`和`LiveData`的相同点:
* 永远有一个值.
* 只有一个值.
* 支持多个观察者.
* 在订阅的瞬间, replay最新的值.

有一点点不同:
* `StateFlow`需要一个初始值.
* `LiveData`会自动解绑, flow要达到相同效果, collect要在`Lifecycle.repeatOnLifecycle`里.


### 使用Flow注意事项
在UI层收集的时候注意要用`repeatOnLifecycle`:
```kotlin
class LatestNewsActivity : AppCompatActivity() {
    private val latestNewsViewModel = // getViewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        //...
        // Start a coroutine in the lifecycle scope
        lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // Note that this happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                latestNewsViewModel.uiState.collect { uiState ->
                    // New value received
                    when (uiState) {
                        is LatestNewsUiState.Success -> showFavoriteNews(uiState.news)
                        is LatestNewsUiState.Error -> showError(uiState.exception)
                    }
                }
            }
        }
    }
}
```

这里有个扩展方法也挺好的:
```kotlin
class FlowObserver<T> (
    lifecycleOwner: LifecycleOwner,
    private val flow: Flow<T>,
    private val collector: suspend (T) -> Unit
) {

    private var job: Job? = null

    init {
        lifecycleOwner.lifecycle.addObserver(LifecycleEventObserver {
                source: LifecycleOwner, event: Lifecycle.Event ->
            when (event) {
                Lifecycle.Event.ON_START -> {
                    job = source.lifecycleScope.launch {
                        flow.collect { collector(it) }
                    }
                }
                Lifecycle.Event.ON_STOP -> {
                    job?.cancel()
                    job = null
                }
                else -> { }
            }
        })
    }
}


inline fun <reified T> Flow<T>.observeOnLifecycle(
    lifecycleOwner: LifecycleOwner,
    noinline collector: suspend (T) -> Unit
) = FlowObserver(lifecycleOwner, this, collector)

inline fun <reified T> Flow<T>.observeInLifecycle(
    lifecycleOwner: LifecycleOwner
) = FlowObserver(lifecycleOwner, this, {})
```
TODO: 看一下官方的`repeatOnLifecycle`是不是就是这个意思.

### `shareIn`和`stateIn`
`shareIn`可以保证只有一个数据源被创造, 并且被所有collectors收集.
比如:
```kotlin
class LocationRepository(
    private val locationDataSource: LocationDataSource,
    private val externalScope: CoroutineScope
) {
    val locations: Flow<Location> = 
        locationDataSource.locationsSource.shareIn(externalScope, WhileSubscribed())
}
```
`WhileSubscribed`这个策略是说, 当无人观测时, 上游的flow就被取消.

实际使用时可以用`WhileSubscribed(5000)`, 让上游的flow即便在无人观测的情况下, 也能继续保持5秒.
这样可以在某些情况(比如旋转屏幕)时避免重建上游资源, 适用于上游资源创建起来很expensive的情况.

如果我们的需求是, 永远保持一个最新的cache值.
```kotlin

class LocationRepository(
    private val locationDataSource: LocationDataSource,
    private val externalScope: CoroutineScope
) {
    val locations: Flow<Location> = 
        locationDataSource.locationsSource.stateIn(externalScope, WhileSubscribed(), EmptyLocation)
}
```
`Flow.stateIn`将会缓存最后一个值, 并且有新的collector时, 将这个最新值传给它.

### `shareIn`, `stateIn`使用注意事项
永远不要在方法里面调用`shareIn`和`stateIn`, 因为方法每次被调用, 它们都会创建新的流.
这些流没有被复用, 会存在内存里面, 直到scope被取消或者没有引用时被GC.

推荐的使用方式是在property上用:

```kotlin
class UserRepository(
    private val userLocalDataSource: UserLocalDataSource,
    private val externalScope: CoroutineScope
) {
    // DO NOT USE shareIn or stateIn in a function like this.
    // It creates a new SharedFlow/StateFlow per invocation which is not reused!
    fun getUser(): Flow<User> =
        userLocalDataSource.getUser()
            .shareIn(externalScope, WhileSubscribed())    

    // DO USE shareIn or stateIn in a property
    val user: Flow<User> = 
        userLocalDataSource.getUser().shareIn(externalScope, WhileSubscribed())
}
```

### StateFlow使用总结
从ViewModel暴露数据到UI, 用`StateFlow`的两种方式:
1. 暴露一个StateFlow属性, 用`WhileSubscribed`加上一个timeout.
```kotlin
class MyViewModel(...) : ViewModel() {
    val result = userId.mapLatest { newUserId ->
        repository.observeItem(newUserId)
    }.stateIn(
        scope = viewModelScope, 
        started = WhileSubscribed(5000), 
        initialValue = Result.Loading
    )
}
```
2. 用`repeatOnLifecycle`收集.
```kotlin
onCreateView(...) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.lifecycle.repeatOnLifecycle(STARTED) {
            myViewModel.myUiState.collect { ... }
        }
    }
}
```

其他的组合都会保持上游的活跃, 浪费资源:
* 用`WhileSubscribed`暴露属性, 在`lifecycleScope.launch/launchWhenX`里收集.
* 通过`Lazily/Eagerly`暴露, 用`repeatOnLifecycle`收集.


## References
* [Kotlin flows on Android](https://developer.android.com/kotlin/flow)
* [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
* [A safer way to collect flows from Android UIs](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)
* [Things to know about Flow’s shareIn and stateIn operators](https://medium.com/androiddevelopers/things-to-know-about-flows-sharein-and-statein-operators-20e6ccb2bc74)
* [Shared flows, broadcast channels](https://elizarov.medium.com/shared-flows-broadcast-channels-899b675e805c)
* [Kotlin SharedFlow or: How I learned to stop using RxJava and love the Flow](https://proandroiddev.com/kotlin-sharedflow-or-how-i-learned-to-stop-using-rxjava-and-love-the-flow-e1b59d211715)
* [Migrating from LiveData to Kotlin’s Flow](https://medium.com/androiddevelopers/migrating-from-livedata-to-kotlins-flow-379292f419fb)
* [Substituting Android’s LiveData: StateFlow or SharedFlow?](https://proandroiddev.com/should-we-choose-kotlins-stateflow-or-sharedflow-to-substitute-for-android-s-livedata-2d69f2bd6fa5)
* [Learning State & Shared Flows with Unit Tests](https://codingwithmohit.com/coroutines/learning-shared-and-state-flows-with-tests/)
* [Reactive Streams on Kotlin: SharedFlow and StateFlow](https://www.raywenderlich.com/22030171-reactive-streams-on-kotlin-sharedflow-and-stateflow)