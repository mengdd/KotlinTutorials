# Coroutines Channels
Java中的多线程通信, 总会涉及到共享状态(shared mutable state)的读写, 有同步, 死锁等问题要处理.

协程中的Channel用于协程间的通信, 它的宗旨是:
```
Do not communicate by sharing memory; instead, share memory by communicating.
```

## Channel basics
channels用于协程间的通信, 允许我们在不同的协程间传递数据(a stream of values).

### 生产者-消费者模式
发送数据到channel的协程被称为`producer`, 从channel接受数据的协程被称为`consumer`.

生产: send, produce.
消费: receive, consume.

当需要的时候, 多个协程可以向同一个channel发送数据, 一个channel的数据也可以被多个协程接收.

当多个协程从同一个channel接收数据的时候, 每个元素仅被其中一个consumer消费一次. 处理元素会自动将其从channel里删除.

### Channel的特点
`Channel`在概念上有点类似于`BlockingQueue`, 元素从一端被加入, 从另一端被消费. 关键的区别在于, 读写的方法不是blocking的, 而是suspending的. 
在为空或为满时. channel可以suspend它的`send`和`receive`操作.

## Channel的关闭和迭代
Channel可以被关闭, 说明没有更多的元素了.
取消producer协程也会关闭channel.

在receiver端有一种方便的方式来接收: 用`for`迭代.

看这个例子:
```
fun main() = runBlocking<Unit> {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x)
        channel.close() // we're done sending
    }
// here we print received values using `for` loop (until the channel is closed)
    for (y in channel) println(y)
    println("Done!")
}
```
运行后会输出:
```
1
2
3
4
5
Done!

Process finished with exit code 0
```

如果注释掉`channel.close()`就会变成:
```
1
2
3
4
5
```
Done没有被输出, 程序也没有退出, 这是因为接受者协程还在一直等待.

## 不同的Channel类型
库中定义了多个channel类型, 它们的主要区别在于: 
* 内部可以存储的元素数量; 
* `send`是否可以被挂起.

所有channel类型的`receive`方法都是同样的行为: 如果channel不为空, 接收一个元素, 否则挂起.

Channel的不同类型:
* Rendezvous channel: 0尺寸buffer, `send`和`receive`要meet on time, 否则挂起. (默认类型).
* Unlimited channel: 无限元素, `send`不被挂起.
* Buffered channel: 指定大小, 满了之后`send`挂起.
* Conflated channel: 新元素会覆盖旧元素, receiver只会得到最新元素, `send`永不挂起.

创建channel:
```
val rendezvousChannel = Channel<String>()
val bufferedChannel = Channel<String>(10)
val conflatedChannel = Channel<String>(CONFLATED)
val unlimitedChannel = Channel<String>(UNLIMITED)
```
默认是Rendezvous channel.



## 练习: 分析代码输出
看这段代码:
```
fun main() = runBlocking<Unit> {
    val channel = Channel<String>()
    launch {
        channel.send("A1")
        channel.send("A2")
        log("A done")
    }
    launch {
        channel.send("B1")
        log("B done")
    }
    launch {
        repeat(3) {
            val x = channel.receive()
            log(x)
        }
    }
}

fun log(message: Any?) {
    println("[${Thread.currentThread().name}] $message")
}
```
这段代码创建了一个channel, 传递String类型的元素.
两个producder协程, 分别向channel发送不同的字符串, 发送完毕后打印各自的"done".
一个receiver协程, 接收channel中的3个元素并打印.

程序的运行输出结果会是怎样呢?

记得在Configurations中加上VM options: `-Dkotlinx.coroutines.debug`. 可以看到协程信息.


答案揭晓:
```
[main @coroutine#4] A1
[main @coroutine#4] B1
[main @coroutine#2] A done
[main @coroutine#3] B done
[main @coroutine#4] A2
```

答对了吗? 

为什么会是这样呢? 原因主要有两点:
* 这里创建的channel是默认的Rendezvous类型, 没有buffer, send和receive必须要meet, 否则挂起.
* 两个producer和receiver协程都运行在同一个线程上, ready to be resumed也只是加入了一个等待队列, resume要按顺序来.

这个例子在[Introduction to Coroutines and Channels](https://play.kotlinlang.org/hands-on/Introduction%20to%20Coroutines%20and%20Channels/08_Channels)中有一个视频解说.


另外, 官方文档中还有一个ping-pang的例子, 为了说明Channels are fair.


## 参考
* [官方文档: Channels](https://kotlinlang.org/docs/reference/coroutines/channels.html)
* [Introduction to Coroutines and Channels](https://play.kotlinlang.org/hands-on/Introduction%20to%20Coroutines%20and%20Channels/08_Channels)
* [Github: Coroutines Guide](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md)
* [Kotlin: Diving in to Coroutines and Channels](https://proandroiddev.com/kotlin-coroutines-channels-csp-android-db441400965f)