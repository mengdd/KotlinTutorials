# Kotlin的属性和代理属性
## 基础知识
属性的声明语法:
```
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```
其中`[]`包含的部分都是可选的.

getter/setter如果不写就是默认实现. 类型如果不能推断出来则不能省略.

只读属性用`val`声明, 不允许有setter.

### getter & setter
可以自定义getter和setter, 即`get()`和`set(value)`方法:

```
var stringRepresentation: String
    get() = this.toString()
    set(value) {
        setDataFromString(value) // parses the string and assigns values to other properties
    }
```
根据惯例, setter的参数名是`value`.

`get()/set()`比较常见的用途有: 格式化, 数据转换, 可见性封装, 只读控制, 输入验证等.

在Java中, 如果在getter/setter方法中写了一些自定义逻辑, 那么一种容易出错的情形是: 访问时不小心使用了字段本身而不是用getter/setter, 从而绕过了这些逻辑.

在Kotlin中则没有这种烦恼, 访问和修改属性都只有一种方法, 每次访问都是通过`get()/set()`方法.

可以添加注解或者可见性修饰, 如果仍然是默认实现可以省略方法体:
```
var setterVisibility: String = "abc"
    private set // the setter is private and has the default implementation

var setterWithAnnotation: Any? = null
    @Inject set // annotate the setter with Inject
```

### Backing Fields
Kotlin中是不能直接定义fields的, 定义出来的都是property.

当property需要一个backing field时, Kotlin会自动提供. 在getter/setter中用`field`标识符访问.
```
var counter = 0 // Note: the initializer assigns the backing field directly
    set(value) {
        if (value >= 0) field = value
    }
```

`field`的存在时很有必要的. 前面说过, 访问属性一定是通过`get()`, 所以如果这样写:
```
var speed: String = "0"
    get() = "$speed km/h"
```
会抛出`StackOverflowError`, 这是因为这里发生了递归调用.

正确的写法是这样:
```
var speed: String = "0"
    get() = "$field km/h"
```

请注意getter/setter中不一定需要用到默认实现, 自定义的getter/setter中可能没有使用`field`标识.

```
val isEmpty: Boolean
    get() = this.size == 0
```
这种是没有backing field的.

可以定义一些属性, 其`get()`返回由其他属性计算得到的值.


### Backing properties
如果你需要的跟这个默认的backing field不符, 你也可以用一个"backing property".

说白了就是一个`private`的property.
```
private var _table: Map<String, Int>? = null
public val table: Map<String, Int>
    get() {
        if (_table == null) {
            _table = HashMap() // Type parameters are inferred
        }
        return _table ?: throw AssertionError("Set to null by another thread")
    }
```
### Overriding properties
properties可以被override, 跟方法一样, 子类可以覆盖父类.

可以用`var`来覆盖`val`, 但是反过来不行.

## lateinit
不可为null的属性被声明后, 可能不能直接初始化, 在构造里也还太早.

常见的比如Android中的各种View变量, 或者是需要通过依赖注入来初始化的一些字段.

如果声明一个nullable的类型, 可以初始化为null,  但之后每次用到都要做null判断, 太不方便了.

`lateinit`修饰符就是用来解决这个问题的.

`lateinit`使用时:
* 只能修饰`var`, 不能修饰`val`.
* 不允许属性带声明初始化语句.
* 不允许属性是nullable的类型.
* 不允许属性是primitive类型.

这些要求都是显而易见的, 没写对的时候编译器都会报相应的错误提示.

如果一个property被标记为`lateinit`, 但是使用的时候还没有被赋值, 就会抛出异常: `kotlin.UninitializedPropertyAccessException: lateinit property XXX has not been initialized`.

如果想要检查是否被初始化了, 可以用`.isInitialized`. 注意调用的时候属性前面要加`::`.

但是注意文档里写:
```
This check is only available for the properties that are lexically accessible, i.e. declared in the same type or in one of the outer types, or at top level in the same file.
```
说明了它的调用条件. 

举个例子, 这样是不行的:
```
fun main() {
    val someClass = PropertyDemo()
    if (someClass::propertyA.isInitialized) {
        println(someClass.propertyA)
    }
}

class PropertyDemo {
    lateinit var propertyA: String
}
```
`isInitialized`会被红线, 会报错:`Error: Kotlin: Backing field of 'var propertyA: String' is not accessible at this point`.

可以这样改:
```
fun main() {
    val someClass = PropertyDemo()
    if (someClass.isPropertyAInitialized()) {
        println(someClass.propertyA)
    }
}

class PropertyDemo {
    lateinit var propertyA: String
    fun isPropertyAInitialized() = ::propertyA.isInitialized
}
```
封装一个方法, 把`.isInitialized`的访问放在类内部.


## Delegated Properties
属性有只管读写的默认用法, 也有自定义getter/setter的定制用法, 在这两者中间, 还有一些比较通用的套路用法:
* lazy属性: 第一次使用的时候才初始化.
* observable属性: 属性一经改变, 自动通知listeners.
* 存储在map中的属性.

Kotlin用delegated properties来提供这些支持. 这样我们可以抽取出通用的部分, 供多个类共享.

### 代理属性语法
代理属性的语法是:
```
val/var <property name>: <Type> by <expression>
```
`by`之后的表达式就叫代理. 属性的`get()`/`set()`会被代理到`getValue()`和`setValue()`方法.

一个例子:
```
class DelegateExample {
    var myProperty: String by MyDelegate()
}

class MyDelegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}
```

从Kotlin 1.1开始, 代理属性可以在函数或者代码块内, 不是非得是类成员属性.

幸运的是, Kotlin标准库已经为常用的代理情形提供了工厂方法, 下面来看一下.

### 懒加载: Lazy
应用场景: 属性的计算可能比较复杂和耗时, 或者并没有一个合适的初始化时机, 于是想要在第一次使用的时候初始化, 保存属性值, 后续调用直接使用计算结果.

```
fun main() {
    println(lazyValue)
    println(lazyValue)
    println(lazyValue)
}

val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}
```
这个例子打印出1个"computed!"和3个"Hello".
只有第一次访问的时候会执行lambda表达式, 后续的访问都只返回结果.

注意这里默认的线程安全模式: `LazyThreadSafetyMode.SYNCHRONIZED: Locks are used to ensure that only a single thread can initialize the [Lazy] instance.`

### 可观察属性: Observable
应用场景: 观察者模式, 想要在属性发生变化的时候通知观察者.
`Delegates.observable()`有两个参数, 一个是初始值, 一个是onChange的lambda, 其参数包含了属性, 新旧值.

例子:
```
class User {
    var name: String by Delegates.observable("<no name>") {
        prop, old, new ->
        println("$old -> $new")
    }
}

fun main() {
    val user = User()
    user.name = "first"
    user.name = "second"
}
```
输出:
```
<no name> -> first
first -> second
```

### 有限制的赋值: vetoable
前面我们写自定义setter的时候有一种情况是要验证输入的有效性, 过滤无效数据. 利用代理属性也可以做这件事: 用`Delegates.vetoable()`.

和`Delegates.observable()`类似, 也是两个参数: 初始值和`onChange`函数, 不同的是: 
* `onChange`此时返回`Boolean`, 只有为true的情况才会成功赋值. 
* `onChange`是在变化发生之前调用, 而`observable()`的onChange是在变化之后调用.

例子:
```
fun main() {
    val child = Child()

    println("try a negative age")
    child.age = -3
    println("age is: ${child.age}")
    println("try a positive age")
    child.age = 5
    println("age is: ${child.age}")
}

class Child {
    var age: Int by Delegates.vetoable(0) { property, oldValue, newValue ->
        println("${property.name}: $oldValue -> $newValue")
        newValue > 0
    }
}
```
输出:
```
try a negative age
age: 0 -> -3
age is: 0
try a positive age
age: 0 -> 5
age is: 5
```
可见-3的值被拒绝了, 并没有赋值成功.

### Map
应用场景举例: 有时候我们的API会设计把某些字段打包用一个map返回, 来达到一个动态个性化配置的效果.

比如定义类:
```
class User(map: Map<String, Any?>) {
    val name by map
    val age by map
}
```
它的两个属性都是从map中拿的, 属性名就是key.

构造这个类时传入map即可:
```
val user = User(
    mapOf(
        "name" to "John Doe",
        "age" to 25
    )
)
```

如果是`MutableMap`, 那么属性是`var`.


### 自定义代理属性
如果标准库提供的这些代理属性工具都难以满足你的需求, 那么你也可以自定义.

有两个接口:`ReadOnlyProperty`和`ReadWriteProperty`, 分别对应只读的属性和可读写的属性.

这两个接口只是为了方便, 并不是语法要求.

要实现自定义代理可以什么接口都不实现. 看代理属性的语法部分的例子, 只要`getValue()`和`setValue()`方法签名符合即可.

## 思考
* 自定义getter/setter和delegated properties有什么不同呢? -> getter/setter只局限在单个类中, 使用了代理之后, 抽取了通用的逻辑, 可以实现复用. 想象如果自己实现一个lazy的`get()`方法, 然后到处重复它?(lame)


## 参考
* [Properties and Fields](https://kotlinlang.org/docs/reference/properties.html)
* [Property References](https://kotlinlang.org/docs/reference/reflection.html#property-references)
* [Delegated Properties](https://kotlinlang.org/docs/reference/delegated-properties.html)