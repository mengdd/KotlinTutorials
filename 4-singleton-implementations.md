# Kotlin中的单例实现
单例模式需要保证类只存在一个实例, 通常用来节约资源或者保证一致的状态.

单例的实现首先主要是通过私有构造, 保证了类不能在外部被随意实例化.

在Java中实现单例会利用静态方法, 来返回唯一的对象, Kotlin中没有静态了, 写法是什么样呢?

## Java中的单例实现
我们先来回忆一下Java中的几种实现.

方法1:
```java
/**
 * 单例实现方法1:
 * <p>
 * 缺点: 多线程的情况存在问题, 可能会创建出多个对象.
 */
public class Singleton1 {

    private static Singleton1 instance;

    private Singleton1() {
    }

    public static Singleton1 getInstance() {
        if (instance == null) {
            instance = new Singleton1();
        }
        return instance;
    }
}
```
这种方法是最简单的, 什么也没考虑, 就需要实例的时候检查是否已经被创建, 如果没有就创建一个实例, 然后返回这个实例.

在多线程使用的时候, 有可能会创建多个实例, 即单例模式本身无法保证.

方法2:
```java
/**
 * 单例实现方法2: 将实例化操作提前.
 * 在静态初始化的时候创建实例, 保证线程安全.
 * <p>
 * 缺点: 如果从来未使用过, 初始化是一种浪费.
 */
public class Singleton2 {

    private static Singleton2 instance = new Singleton2();

    private Singleton2() {
    }

    public static Singleton2 getInstance() {
        return instance;
    }
}
```
这种实现方式解决了多线程的问题, 采用的方法是在静态初始化的时候就把实例创建好.
缺点就是如果我从来也没有用到过这个实例, 那么就白创建了.

方法3:
```java
/**
 * 单例实现方法3: 用synchronized关键字保证线程安全.
 * <p>
 * 缺点: 会降低性能. 即便对象已经被创建, 仍然多次进行同步.
 */
public class Singleton3 {

    private static Singleton3 instance;

    private Singleton3() {
    }

    public static synchronized Singleton3 getInstance() {
        if (instance == null) {
            instance = new Singleton3();
        }
        return instance;
    }
}
```
用关键字保证了多线程情况下也不会创建多个.
但是创建之后仍然每次都会同步, 有效率问题.

方法4:
```java
/**
 * 单例实现方法4: 双重检查加锁.
 * 只有实例尚未创建才会进行同步.
 * <p>
 * 缺点: 写起来复杂.
 */
public class Singleton4 {

    private volatile static Singleton4 instance; //volatile关键字保证可见性, 字段被访问时用的都是共享值, 而不是缓存拷贝

    private Singleton4() {
    }

    public static Singleton4 getInstance() {
        if (instance == null) {
            synchronized (Singleton4.class) {
                if (instance == null) { //进入块之后会再检查一次
                    instance = new Singleton4();
                }
            }
        }
        return instance;
    }
}
```
这是一个双重检查的锁, 在多线程下保证单例, 同时避免了上面说的效率问题.

## Kotlin单例实现1: Object声明
怎么在Kotlin中实现一个单例呢? 最简单的方法就是用对象声明:
```kotlin
object SingletonOne {
    
}
```
这种实现方法的本质是什么呢? 这里介绍IDE(Android Studio或Intellij)的一个工具:
`Tools -> Kotlin -> Show Kotlin Bytecode`, 显示Kotlin字节码之后, 该窗口左上角还有一个`Decompile`按钮, 点击之后显示:
```
public final class SingletonOne {
   public static final SingletonOne INSTANCE;

   private SingletonOne() {
   }

   static {
      SingletonOne var0 = new SingletonOne();
      INSTANCE = var0;
   }
}
```
怎样, 反编译后发现这不就是上面的第二种实现方式吗, 只不过这里`INSTANCE`域是public的, 所以没有提供getInstance方法.

使用这样的单例时, 直接用类名就可以访问其方法:
```kotlin
object SingletonOneWithMethod {
    private const val age = 20
    fun foo() {
        println(this.hashCode().toString() + " with age $age")
    }
}

fun main() {
    SingletonOneWithMethod.foo()
}
```

## Kotlin单例实现2: class + companion object
单例实现的第二种方法:
```kotlin
class SingletonTwo private constructor() {
    companion object {
        private var instance: SingletonTwo? = null

        fun getInstance() =
            instance ?: SingletonTwo().also { instance = it }
    }
}
```
用的是class, 把构造声明为私有.
其中`getInstance()`方法写得稍微有点炫酷哈, 其实它要做的事情就是检查是否为null, 如果为null则创建实例, 不为null则直接返回. 所以它对应的是Java实现1.

这种实现方法比上一种好在哪里呢?
首先, 它是第一次用到的时候才初始化的, 如果没用到就永远也不会初始化. 其次, 它可以在构造中传入参数.
```kotlin
class SingletonTwoWithArguments private constructor(
    private val name: String,
    private val age: Int
) {
    companion object {
        private var instance: SingletonTwoWithArguments? = null

        fun getInstance(name: String, age: Int) =
            instance ?: SingletonTwoWithArguments(name, age).also { instance = it }
    }

    override fun toString(): String {
        return "SingletonTwoWithArguments${hashCode()}(name='$name', age=$age)"
    }
}

fun main() {
    val instance1 = SingletonTwoWithArguments.getInstance("hello", 1)
    println(instance1)
    val instance2 = SingletonTwoWithArguments.getInstance("world", 2)
    println(instance2)
    val instance3 = SingletonTwoWithArguments.getInstance("ddmeng", 30)
    println(instance3)
}
```
这里的输出结果永远是第一个创建的实例.即"hello, 1".


但是这和Java的同等实现1一样, 它在多线程情况下有可能会创建多个实例.

## Kotlin单例实现3: 为了线程安全, 加同步
这是对上一个实现的改进, 把方法改为同步的, 对应于Java的实现3.
```kotlin
class SingletonThree private constructor() {
    companion object {
        private var instance: SingletonThree? = null

        @Synchronized
        fun getInstance() =
            instance ?: SingletonThree().also { instance = it }
    }
}
```
缺点也是一样的, 实例已经创建后还是会同步, 所以有效率问题.

## Kotlin单例实现4: 双重检查加锁
再对上面的实现做个改进:
```kotlin
class SingletonFour private constructor() {
    companion object {
        @Volatile
        private var instance: SingletonFour? = null

        fun getInstance() =
            instance ?: synchronized(this) {
                instance ?: SingletonFour().also { instance = it }
            }
    }
}
```
这里做了double check, 只有实例为空时才进入同步块, 提升了效率.
注意这里不要忘了字段需要是`@Volatile`的.

这种实现可以参见Google Sample中的一个实例:
[sunflower: GardenPlantingRepositor](https://github.com/googlesamples/android-sunflower/blob/master/app/src/main/java/com/google/samples/apps/sunflower/data/GardenPlantingRepository.kt).
双重检查加锁, 带一个构造参数.

## Kotlin单例实现5: lazy
这里用到一个Kotlin的特性:
[lazy properties](https://kotlinlang.org/docs/reference/delegated-properties.html).

`lazy()`接收一个lambda作为初始化方法, 只有第一次访问的时候会被调用, 然后结果就被保存下来了, 之后的访问会直接返回这个结果. 默认的线程安全模式是`LazyThreadSafetyMode.SYNCHRONIZED`.

使用lazy来实现单例:
```kotlin
class SingletonFive private constructor() {
    companion object {
        val instance: SingletonFive by lazy {
            println("lazy body")
            SingletonFive()
        }
    }
}

fun main() {
    val instance1 = SingletonFive.instance
    println(instance1)
    val instance2 = SingletonFive.instance
    println(instance2)
    val instance3 = SingletonFive.instance
    println(instance3)
}
```
从执行结果可以看出"lazy body"只被输出了一次, 并且实例是同一个.

## Bonus: 实现5: 用Dagger来实现单例
[Dagger](https://github.com/google/dagger)是Android开发中不可或缺的一个好工具. 

Dagger的使用这里不展开介绍了, 只说说单例相关的.

Dagger中的`@Singleton`是一个常用的scope注解. 
它的用法:
* 标记在component上作为一个scope标记.
* 标记在依赖上(类名或module中的provide方法上), 说明这个依赖在该scope下是一个单例. 

看上去只用简单的两个注解, dagger会帮我们做完其他的事情, 保证只有一个实例.

通常我会把我的主component用`@Singleton`标记, 然后把所有全局单例用`@Singleton`标记.

除此之外我们还可以定义其他特定scope下的单例, 比如给Activity的component定义一个scope, 让一些依赖成为这个scope意义下的单例.

