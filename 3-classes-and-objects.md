# Kotlin中的类和对象
Kotlin中的类关键字仍然是`class`, 但是创建类的实例不需要`new`.

## 构造函数
构造函数分为: primary constructor(一个)和secondary constructor(一个或多个).

如果一个非抽象类自己没有声明任何构造器, 它将会生成一个无参数的主构造, 可见性为public.

### 主构造 Primary Constructor
Primary constructor写在类名后面, 作为class header:
```
class Person constructor(firstName: String) {
//... 
}
```

如果没有注解和可见性修饰符, 关键字`constructor`是可以省略的:
```
class Person2(firstName: String) {
//... 
}
```

### init代码块和属性初始化代码
Primary constructor是不能包含任何代码的, 如果需要初始化的代码, 可以放在init块中.

在实例的初始化阶段, init块和属性初始化的执行顺序和它们在body中出现的顺序一致.

```
class InitOrderDemo(name: String) {
    val firstProperty = "First property: $name".also(::println)

    init {
        println("First initializer block that prints $name")
    }

    val secondProperty = "Second property: ${name.length}".also(::println)

    init {
        println("Second initializer block that prints ${name.length}")
    }
}
```
这个类被实例化的时候, print的顺序和代码中的顺序一致.

这个例子也可以看出, 主构造函数中传入的参数在init块和属性初始化代码中是可以直接使用的.

实际上, 对于主构造中传入的参数是可以直接声明为属性并初始化的.
```
class PersonWithProperties(val firstName: String, val lastName: String, var age: Int) {
//    ... 
}
```
可以是`val`也可以是`var`.

### 次构造 Secondary Constructors
次构造是用constructor关键字标记, 写在body里的构造.

如果有主构造, 次构造需要代理到主构造, 用this关键字.

注意init块是和主构造关联的, 所以会在次构造代码之前执行.

即便没有声明主构造, 这个代理也是隐式发生的, 所以init块仍然会执行.
```
class Constructors {
    init {
        println("Init block")
    }

    constructor(i: Int) {
        println("Constructor")
    }
}
```
创建实例, 输出:
```
Init block
Constructor
```

## 继承
继承用`:`.
方法覆写的时候`override`关键字是必须的.

Kotlin中默认是不鼓励继承的:
* 类需要显示声明为`open`才可以被继承.
* 方法要显示声明为`open`才可以被覆写.

抽象类(abstract)默认是`open`的.
一个已经标记为`override`的方法是`open`的, 如果想要禁止它被进一步覆写, 可以在前面加上`final`.

属性也可以被覆盖, 同方法类似, 基类需要标记`open`, 子类标记`override`.

注意`val`可以被覆写成`var`, 但是反过来却不行.
```
interface Foo {
    val count: Int
}

class Bar1(override val count: Int) : Foo

class Bar2 : Foo {
    override var count: Int = 0
}
```

注意, 由于初始化顺序问题. 在基类的构造, init块和属性初始化中, 不要使用open的成员.

## 对象表达式和对象声明
Java中的匿名内部类, 在Kotlin中用对象表达式(expression)和对象声明(declaration).

表达式和声明的区别:
声明不可以被用在=的右边.

### 对象表达式 object expression
用`objec`关键字可以创建一个匿名类的对象.

这个匿名类可以继承一个或多个基类:
```
open class A(x: Int) {
    public open val y: Int = x
}

interface B {
// ...
}

val ab: A = object : A(1), B {
    override val y = 15
}
```
也可以没有基类:
```
fun foo() {
    val adHoc = object {
        var x: Int = 0
        var y: Int = 0
    }
    print(adHoc.x + adHoc.y)
}
```
### 对象声明 object declaration
在Kotlin中可以用`object`来声明一个单例.
```
// singleton
object DataProviderManager {
    fun doSomething() {
    }
}

fun main() {
    DataProviderManager.doSomething()
}
```
对象声明的初始化是线程安全的.
使用的时候直接用类名即可调用它的方法.

### 伴生对象 Companion Objects
在类里面写的对象声明可以用`companion`关键字标记, 表示伴生对象.

Kotlin中的类并没有静态方法.
在大多数情况下, 推荐使用包下的方法.

如果在类中声明一个伴生对象, 就可以像Java中的静态方法一样, 用`类名.方法名`调用方法.
```
class MyClass {
    companion object Factory {
        fun create(): MyClass = MyClass()
    }
}

fun main() {
    MyClass.create()
}
```

## Model类神器: data class
model类可以用`data`来标记为data class.
```
data class User(val name: String, val age: Int)
```
编译器会根据**在主构造中声明的所有属性**, 自动生成需要的方法, 包括`equals()/hashCode()`, `toString()`, `componentN()`和`copy()`.

为了让生成的代码有意义, 需要满足这些条件:
* 主构造至少要有一个参数.
* 所有的主构造参数都要被标记为`val`或`var`.
* data class不能为abstract, open, sealed或inner.
* Kolin 1.1之前, 还只能实现接口.

注意: 
* 如果`equals()`, `hashCode()`或 `toString()`有显式的实现, 或基类有`final`版本, 这三个方法将不会被生成, 而使用现有版本.
* 如果需要生成的类有无参数的构造器, 那么所有的属性都需要指定一个默认值.
```
data class UserWithDefaults(val name: String = "", val age: Int = 0)
```

## 参考
* [Classes](https://kotlinlang.org/docs/reference/classes.html)
* [Object Expressions and Declarations](https://kotlinlang.org/docs/reference/object-declarations.html)
* [Data Classes](https://kotlinlang.org/docs/reference/data-classes.html)