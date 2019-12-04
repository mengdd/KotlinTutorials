# Coroutines在Android中的实践
前面两篇文章讲了协程的基础知识和协程的通信. 举的例子可能离实际的应用代码比较遥远. 

这篇我们就从Android应用的角度, 看看实践中都有哪些地方可以用到协程.

## Coroutines的用途
Coroutines在Android中可以帮我们做什么:
* 取代callbacks, 简化代码, 改善可读性.
* 保证Main safety.
* 结构化管理和取消任务, 避免泄漏.

这有一个例子:
```
suspend fun fetchDocs() {                      // Dispatchers.Main
    val result = get("developer.android.com")  // Dispatchers.Main
    show(result)                               // Dispatchers.Main
}

suspend fun get(url: String) =                 // Dispatchers.Main
    withContext(Dispatchers.IO) {              // Dispatchers.IO (main-safety block)
        /* perform network IO here */          // Dispatchers.IO (main-safety block)
    }                                          // Dispatchers.Main
}
```
这里`get`是一个`suspend`方法, 只能在另一个`suspend`方法或者在一个协程中调用.

`get`方法在主线程被调用, 它在开始请求之前suspend了协程, 当请求返回, 这个方法会resume协程, 回到主线程. 网络请求不会block主线程.

### main-safety是如何保证的呢?
dispatcher决定了协程在什么线程上执行. 每个协程都有dispatcher. 协程suspend自己, dispatcher负责resume它们.
* `Dispatchers.Main`: 主线程: UI交互, 更新`LiveData`, 调用`suspend`方法等.
* `Dispatchers.IO`: IO操作, 数据库操作, 读写文件, 网路请求.
* `Dispatchers.Default`: 主线程之外的计算任务(CPU-intensive work), 排序, 解析JSON等.

一个好的实践是使用`withContext()`来确保每个方法都是main-safe的, 调用者可以在主线程随意调用, 不用关心里面的代码到底是哪个线程的.

### 管理协程
之前讲Scope和Structured Concurrency的时候提过, scope最典型的应用就是按照对象的生命周期, 自动管理其中的协程, 及时取消, 避免泄漏和冗余操作.

在协程之中再启动新的协程, 父子协程是共享scope的, 也即scope会track其中所有的协程.

协程被取消会抛出`CancellationException`.

`coroutineScope`和`supervisorScope`可以用来在suspend方法中启动协程. Structured concurrency保证: 当一个suspend函数返回时, 它的所有工作都执行完毕.

它们两者的区别是: 当子协程发生错误的时候, `coroutineScope`会取消scope中的所有的子协程, 而`supervisorScope`不会取消没有发生错误的其他子协程.


## Activity/Fragment & Coroutines
在Android中, 可以把一个屏幕(Activity/Fragment)和一个`CoroutineScope`关联, 这样在Activity或Fragment生命周期结束的时候, 可以取消这个scope下的所有协程, 好避免协程泄漏.

利用`CoroutineScope`来做这件事有两种方法: 创建一个`CoroutineScope`对象和activity的生命周期绑定, 或者让activity实现`CoroutineScope`接口.

方法1: 持有scope引用:
```
class Activity {
    private val mainScope = MainScope()
    
    fun destroy() {
        mainScope.cancel()
    }
}    
```

方法2: 实现接口:
```
class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {
    fun destroy() {
        cancel() // Extension on CoroutineScope
    }
}
```
默认线程可以根据实际的需要指定.
Fragment的实现类似, 这里不再举例.

## ViewModel & Coroutines
Google目前推广的MVVM模式, 由ViewModel来处理逻辑, 在ViewModel中使用协程, 同样也是利用scope来做管理.

ViewModel在屏幕旋转的时候并不会重建, 所以不用担心协程在这个过程中被取消和重新开始.

### 方法1: 自己创建scope
```
private val viewModelJob = Job()

private val uiScope = CoroutineScope(Dispatchers.Main + viewModelJob)
```
默认是在UI线程.
`CoroutineScope`的参数是`CoroutineContext`, 是一个配置属性的集合. 这里指定了dispatcher和job.

在ViewModel被销毁的时候:
```
override fun onCleared() {
    super.onCleared()
    viewModelJob.cancel()
}
```
这里viewModelJob是uiScope的job, 取消了viewModelJob, 所有这个scope下的协程都会被取消.

一般`CoroutineScope`创建的时候会有一个默认的job, 可以这样取消:
```
uiScope.coroutineContext.cancel()
```


### 方法2: 利用`viewModelScope`
如果我们用上面的方法, 我们需要给每个ViewModel都这样写. 为了避免这些boilerplate code, 我们可以用`viewModelScope`. 

注: 要使用viewModelScope需要添加相应的KTX依赖.
* For ViewModelScope, use `androidx.lifecycle:lifecycle-viewmodel-ktx:2.1.0-beta01` or higher.

`viewModelScope`绑定的是`Dispatchers.Main`, 会自动在ViewModel clear的时候自动取消.

用的时候直接用就可以了:
```
class MainViewModel : ViewModel() {
    // Make a network request without blocking the UI thread
    private fun makeNetworkRequest() {
       // launch a coroutine in viewModelScope 
        viewModelScope.launch(Dispatchers.IO) {
            // slowFetch()
        }
    }

    // No need to override onCleared()
}

```
所有的setting up和clearing工作都是库完成的.

## LifecycleScope & Coroutines
每一个[Lifecycle](https://developer.android.com/topic/libraries/architecture/lifecycle)对象都有一个`LifecycleScope`.

同样也需要添加依赖:
* For LifecycleScope, use `androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha01` or higher.


要访问`CoroutineScope`可以用`lifecycle.coroutineScope`或者`lifecycleOwner.lifecycleScope`属性.

比如:
```
activity.lifecycleScope.launch {}
fragment.lifecycleScope.launch {}
fragment.viewLifecycleOwner.launch {}
```
`lifecycleScope`可以启动协程, 当Lifecycle结束的时候, 任何这个scope中启动的协程都会被取消.

这比较适合于处理一些带delay的UI操作, 比如需要用handler.postDelayed的更新UI的操作, 有多个操作的时候嵌套难看, 还容易有泄漏问题.

用了lifecycleScope之后, 既避免了嵌套代码, 又自动处理了取消.

```
lifecycleScope.launch {
    delay(DELAY)
    showFullHint()
    delay(DELAY)
    showSmallHint()
}
```

### LifecycleScope和ViewModelScope
但是LifecycleScope启动的协程却不适合调用repository的方法. 因为它的生命周期和Activity/Fragment是一致的, 太碎片化了, 容易被取消, 造成浪费.

设备旋转时, Activity会被重建, 如果取消请求再重新开始, 会造成一种浪费.

可以把请求放在ViewModel中, UI层重新注册获取结果. `viewModelScope`和`lifecycleScope`可以结合起来使用.

举例: ViewModel这样写:
```
class NoteViewModel: ViewModel {
    val noteDeferred = CompletableDeferred<Note>()
    
    viewModelScope.launch {
        val note = repository.loadNote()
        noteDeferred.complete(note)
    }
    
    suspend fun loadNote(): Note = noteDeferred.await()
}
```

而我们的UI中:
```
fun onCreate() {
    lifecycleScope.launch {
        val note = userViewModel.loadNote()
        updateUI(note)
    }
}
```

这样做之后的好处:
* ViewModel保证了数据请求没有浪费, 屏幕旋转不会重新发起请求.
* lifecycleScope保证了view没有leak.


### 特定生命周期阶段
尽管scope提供了自动取消的方式, 你可能还有一些需求需要限制在更加具体的生命周期内.

比如, 为了做`FragmentTransaction`, 你必须等到`Lifecycle`至少是`STARTED`.

上面的例子中, 如果需要打开一个新的fragment:
```
fun onCreate() {
    lifecycleScope.launch {
        val note = userViewModel.loadNote()
        fragmentManager.beginTransaction()....commit() //IllegalStateException
    }
}
```
很容易发生`IllegalStateException`.

Lifecycle提供了:
`lifecycle.whenCreated`, `lifecycle.whenStarted`, `lifecycle.whenResumed`.

如果没有至少达到所要求的最小生命周期, 在这些块中启动的协程任务, 将会suspend.

所以上面的例子改成这样:
```
fun onCreate() {
    lifecycleScope.launchWhenStarted {
        val note = userViewModel.loadNote()
        fragmentManager.beginTransaction()....commit()
    }
}
```

如果`Lifecycle`对象被销毁(`state==DESTROYED`), 这些when方法中的协程也会被自动取消.

## LiveData & Coroutines
`LiveData`是一个供UI观察的value holder.

`LiveData`的数据可能是异步获得的, 和协程结合:
```
val user: LiveData<User> = liveData {
    val data = database.loadUser() // loadUser is a suspend function.
    emit(data)
}
```
这个例子中的`liveData`是一个builder function, 它调用了读取数据的方法(一个`suspend`方法), 然后用`emit()`来发射结果.

同样也是需要添加依赖的:
* For liveData, use `androidx.lifecycle:lifecycle-livedata-ktx:2.2.0-alpha01` or higher.


实际上使用时, 可以`emit()`多次:
```
val user: LiveData<Result> = liveData {
    emit(Result.loading())
    try {
        emit(Result.success(fetchUser()))
    } catch(ioException: Exception) {
        emit(Result.error(ioException))
    }
}
```
每次`emit()`调用都会suspend这个块, 直到`LiveData`的值在主线程被设置. 

`LiveData`还可以做变换:
```
class MyViewModel: ViewModel() {
    private val userId: LiveData<String> = MutableLiveData()
    val user = userId.switchMap { id ->
        liveData(context = viewModelScope.coroutineContext + Dispatchers.IO) {
            emit(database.loadUserById(id))
        }
    }
}
```

如果数据库的方法返回的类型是LiveData类型, `emit()`方法可以改成`emitSource()`. 例子见: [Use coroutines with LiveData](https://developer.android.com/topic/libraries/architecture/coroutines#livedata).


## 网络/数据库 & Coroutines
根据Architecture Components的构建模式:
* ViewModel负责在主线程启动协程, 清理时取消协程, 收到数据时用`LiveData`传给UI.
* Repository暴露`suspend`方法, 确保方法main-safe.
* 数据库和网络暴露`suspend`方法, 确保方法main-safe. Room和Retrofit都是符合这个pattern的.

Repository暴露`suspend`方法, 是主线程safe的, 如果要对结果做一些heavy的处理, 比如转换计算, 需要用`withContext`自行确定主线程不被阻塞.

### Retrofit & Coroutines
Retrofit从2.6.0开始提供了对协程的支持.

定义方法的时候加上`suspend`关键字:
```
interface GitHubService {
    @GET("orgs/{org}/repos?per_page=100")
    suspend fun getOrgRepos(
        @Path("org") org: String
    ): List<Repo>
}
```
suspend方法进行请求的时候, 不会阻塞线程.
返回值可以直接是结果类型, 或者包一层`Response`:
```
@GET("orgs/{org}/repos?per_page=100")
suspend fun getOrgRepos(
    @Path("org") org: String
): Response<List<Repo>>
```

### Room & Coroutines
Room从2.1.0版本开始提供对协程的支持. 具体就是DAO方法可以是`suspend`的.
```
@Dao
interface UsersDao {
    @Query("SELECT * FROM users")
    suspend fun getUsers(): List<User>

    @Insert
    suspend fun insertUser(user: User)

    @Update
    suspend fun updateUser(user: User)

    @Delete
    suspend fun deleteUser(user: User)
}
```

Room使用自己的dispatcher来确定查询运行在后台线程.
所以你的代码不应该使用`withContext(Dispatchers.IO)`, 会让代码变得复杂并且查询变慢.

更多内容可见: [Room 🔗 Coroutines](https://medium.com/androiddevelopers/room-coroutines-422b786dc4c5).

## WorkManager & Coroutines
WorkManager也有协程版本, 添加`work-runtime-ktx`依赖, 然后改变基类, 以前继承`Worker`, 现在继承`CoroutineWorker`.
比如: 
```
class UploadNotesWorker(...) : CoroutineWorker(...) {
    suspend fun doWork(): Result {
        val newNotes = db.queryNewNotes()
        noteService.uploadNotes(newNotes)
        db.markAsSynced(newNotes)
        return Result.success()
    }
}
```
这段代码其中数据库用`Room`, 网络用`Retrofit`, 这样3个方法都是`suspend`的.

用了协程的版本之后, 取消操作更容易.

更详细的请看: [Threading in CoroutineWorker](https://developer.android.com/topic/libraries/architecture/workmanager/advanced/coroutineworker)


## 异常处理
`suspend`方法中的异常将会resume到调用者.
更一般的, 协程中的错误会通知到它的调用者或者scope.

`launch`和`async`的异常处理不同.
这是因为`async`返回值, 是期待`await`调用的, 所以会持有异常, 在调用`await()`的时候才返回(结果或异常).
所以如果`await()`没有被调用的话, 异常就会被吃了.

## 测试
推荐使用`runBlockingTest`来替换`runBlocking`, 将会利用virtual time, 节省测试时间.

更多关于测试的详细内容见: [kotlinx-coroutines-test](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-test/)

## 参考
* [Codelab: Using Kotlin Coroutines in your Android App](https://codelabs.developers.google.com/codelabs/kotlin-coroutines/#0)
* [Improve app performance with Kotlin coroutines](https://developer.android.com/kotlin/coroutines)
* [Use Kotlin coroutines with Architecture components](https://developer.android.com/topic/libraries/architecture/coroutines)
* [Coroutine Context and Dispatchers](https://kotlinlang.org/docs/reference/coroutines/coroutine-context-and-dispatchers.html)
* [Threading in CoroutineWorker](https://developer.android.com/topic/libraries/architecture/workmanager/advanced/coroutineworker)

博客:
* [Kotlin Coroutines patterns & anti-patterns](https://proandroiddev.com/kotlin-coroutines-patterns-anti-patterns-f9d12984c68e)
* [Coroutines on Android (part II): Getting started](https://medium.com/androiddevelopers/coroutines-on-android-part-ii-getting-started-3bff117176dd)
* [Coroutines On Android (part III): Real work](https://medium.com/androiddevelopers/coroutines-on-android-part-iii-real-work-2ba8a2ec2f45)
* [Part 2 — Coroutine Cancellation and Structured Concurrency](https://proandroiddev.com/part-2-coroutine-cancellation-and-structured-concurrency-2dbc6583c07d)
* [Room 🔗 Coroutines](https://medium.com/androiddevelopers/room-coroutines-422b786dc4c5)

Google的视频:
* [LiveData with Coroutines and Flow (Android Dev Summit '19)](https://www.youtube.com/watch?v=B8ppnjGPAGE&list=PLWz5rJ2EKKc_xXXubDti2eRnIKU0p7wHd&index=4)
* [Understand Kotlin Coroutines on Android (Google I/O'19)](https://www.youtube.com/watch?v=BOHK_w09pVA&t=15s)
