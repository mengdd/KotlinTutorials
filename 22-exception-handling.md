# 协程中的异常处理

## catch不住的exception
看这个代码片段:
```kotlin
fun main() {
    val scope = CoroutineScope(Job())
    try {
        scope.launch {
            throw RuntimeException()
        }
    } catch (e: Exception) {
        println("Caught: $e")
    }

    Thread.sleep(100)
}
```
这里的异常能catch住吗? 为什么?

我这么问了肯定是catch不住的, 但是如何解释呢?

## Parent-Child关系
如果一个coroutine抛出了异常, 它将会把这个exception向上抛给它的parent, 它的parent会做一下三件事情:
- 取消其他所有的children.
- 取消自己.
- 把exception继续向上传递.

### 如果不想让失败取消parent和其他siblings
如果有一些情形, 开启了多个child job, 但是却不想因为其中一个的失败而取消其他, 怎么办?
用`SupervisorJob`.

比如: 
```
val uiScope = CoroutineScope(SupervisorJob())
```

如果你用的是scope builder, 那么用`supervisorScope`.

###



## 不能被catch的exception


## Further Reading

Android官方文档上链接的博客和视频:
- [Exceptions in coroutines](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
- [KotlinConf 2019: Coroutines! Gotta catch 'em all! by Florina Muntenescu & Manuel Vivo](https://www.youtube.com/watch?v=w0kfnydnFWI)

其他:
- [Kotlin Coroutines and Flow - Use Cases on Android](https://github.com/LukasLechnerDev/Kotlin-Coroutines-and-Flow-UseCases-on-Android)
- [Why exception handling with Kotlin Coroutines is so hard and how to successfully master it!](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/)