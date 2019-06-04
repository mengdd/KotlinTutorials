# 为什么要学习Kotlin
这篇文章的来源是看了陈皓的专栏, 里面有一个学习模板, 列出了学习新技术的时候首先要搞清楚的6个问题:
```
## 1.这个技术出现的背景, 初衷, 要达到什么样的目标或是要解决什么样的问题.
## 2.这个技术的优势和劣势分别是什么, 或者说, 这个技术的trade-off是什么.
## 3.这个技术使用的场景.
## 4.技术的组成部分和关键点.
## 5.技术的底层原理和关键实现.
## 6.已有的实现和它之间的对比.
```

## 1.这个技术出现的背景, 初衷, 要达到什么样的目标或是要解决什么样的问题.
历史:
* 2011年7月, JetBrains公开了他们的Kotlin项目, 一个新的运行在JVM上的静态编程语言.
* 2016年Kotlin v1.0发布, 这是第一个官方的稳定版. [Kotlin 1.0 Released](https://blog.jetbrains.com/kotlin/2016/02/kotlin-1-0-released-pragmatic-language-for-jvm-and-android/)
* 2017年5月, [Google I/O 2017](https://android-developers.googleblog.com/2017/05/google-io-2017-empowering-developers-to.html), Google官方宣布了Kotlin作为Android开发的推荐首选语言. 

Kotlin要解决的痛点:
* Java太老(之前Android不支持Java8, 需要一些第三方工具).
* 太啰嗦.
* 容易出错(NPE). 
* 一些方便的工具类或方法需要依赖第三方库比如guava.  

Kotlin更现代化, 更方便简洁, 可以作为更好的一个替代.

Kotlin的目标:
* 成为各种平台, 各种应用开发的统一语言. 包括全站web应用, Android和iOS客户端, 嵌入式/IoT等等. Kotlin/JVM, Kotlin/JS, Kotlin/Native.

## 2.这个技术的优势和劣势分别是什么, 或者说, 这个技术的trade-off是什么.
优势: 
* 更加简洁.
* 更安全: 从类型上处理空安全.
* 和Java良好的互操作性.
* 可以复用Java和JavaScript的库.
* 良好的IDE支持.
* 支持lambda, 函数式编程.
* 很多有用的集合扩展方法.
* 还有很多先进和方便的特性...

缺点:

我在网上找了找Kotlin的缺点, 发现都是一些早期的文章, 现在这些缺点感觉都站不住脚了, 或者都不能算是痛点.

* 需要学习. 团队如果要开始学习短期内可能会影响效率. -> 其实Kotlin上手起来很容易的. 学习一个新语言从来都不应该作为一个难点.
* 担心Kotlin编译时间慢. 这里有个[比较的文章](https://medium.com/keepsafe-engineering/kotlin-vs-java-compilation-speed-e6c174b39b5d). 当时就说明了这不是问题.
* IDE提供的自动转换功能错误多多. -> 其实不太能用到大面积的自动转换. 因为即便是在现有项目上采用Kotlin, 一般也是新赠代码用Kotlin. 毕竟Kotlin和Java可以兼容. 而且即便真的要用自动转换, 熟悉了之后要改的点也就那么几种.
* 社区支持不够. -> 现在社区支持很好.
* 还有一些语言具体方面的不便, 比如没有static, 类默认是不open的等, 见这个[文章](https://medium.com/keepsafe-engineering/kotlin-the-good-the-bad-and-the-ugly-bf5f09b87e6f), 但是这本质上是因为用惯了Java的习惯问题,  用习惯了之后觉得这些都还好, 什么问题都有对应的解决方案.

## 3.这个技术使用的场景.
### Android
[Kotlin for Android](https://kotlinlang.org/docs/reference/android-overview.html)

Kotlin目前是Android App开发的官方推荐语言.


### 服务器端
[Kotlin for server side](https://kotlinlang.org/docs/reference/server-overview.html)

因为Kotlin可以取代Java, 做之前Java做的任何事情. 所以它也可以写服务器端代码.

### 前端
[Kotlin for JavasSript](https://kotlinlang.org/docs/reference/js-overview.html)

Kotlin还可以编译JavaScript, 在浏览器中运行.

### Kotlin/Native
[Kotlin/Native](https://kotlinlang.org/docs/reference/native-overview.html)

把Kotlin编译成本地语言, 不需要虚机也能运行.

### Multiplatform
[Kotlin Multiplatform](https://kotlinlang.org/docs/reference/multiplatform.html)

Multiplatform目前(2019.5)还是一个Kotlin1.2和1.3上的实验项目.

目标是支持: JVM, Android, JavaScript, iOS, Linux, Windows, Mac甚至嵌入式系统, 从而能支持多平台的代码共享.

### iOS
借助于[Intel Multi-OS Engine](https://software.intel.com/en-us/multi-os-engine)等工具可以让Kotlin代码运行在iOS设备上.


### 桌面应用程序
可以使用Kotlin和[TornadoFX](https://github.com/edvin/tornadofx), [JavaFX](https://docs.oracle.com/javase/8/javafx/get-started-tutorial/jfx-overview.htm)一起来构建桌面应用程序.


## 4.技术的组成部分和关键点.
* 基础: 类型, 控制流, 类和对象.
* 函数式: 高阶函数, Lambda.
* 空安全.
* 集合.
* 作用域方法.
* 扩展方法.
* 代理.
* 和Java的互相调用.
* 协程.

Kotlin的文档:
[Kotlin reference](https://kotlinlang.org/docs/reference/)

基本上经常拿出来说的卖点就是空安全, data class, 和Java的互相调用, 方便的集合操作, lambda和高阶函数等, 协程也是一个热点话题, 新的异步任务的写法. 

## 5.技术的底层原理和关键实现.
Kotlin是基于JVM的语言.
通过Kotlin编译器生成的字节码与Java编译的字节码基本相同, 也因此与Java可以完全兼容.

Kotlin与Java编译过程的不同主要在于目标代码生成环节, Kotlin做了更多的工作, 比如生成getter/setter, 修改类属性为final等.

参考: [Kotlin编译过程分析](http://shinelw.com/2017/03/19/kotlin-compiler-process-analysis/)

## 6.已有的实现和它之间的对比.
对于Android开发者来说, Kotlin是为了取代Java而出现的.
它的主要优点: 更简洁, 更安全, 更先进, 更好用.



具体的点可以参见下面的参考资料,
第一个是官方列出的Kotlin和Java的对比, 第二个文章是Swift和Kotlin的对比.

参考:
* [Kotlin官网文档comparison to java](https://kotlinlang.org/docs/reference/comparison-to-java.html)
* [Swift is like Kotlin](http://nilhcem.com/swift-is-like-kotlin/)