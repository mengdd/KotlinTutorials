# Kotlin和Java的互相调用
Kotlin和Java是有互操作性的(Interoperability). Kotlin和Java代码可以互相调用.

为什么在一个项目里这两种语言会同时存在呢?
* 改造Java项目迁移到Kotlin时, 渐进改动就会有两种语言同时存在, 相互调用的情况.
* 即便你选择了一种语言, 很可能也需要用到库是用另一种语言写的. 比如新写一个Kotlin项目, 但是用到的库仍然是Java的.

## Kotlin调用Java
### 空安全
因为Java中的所有引用都是可能为null的. Java声明的类型在Kotlin中被称为`platform types`.

从Java中传过来的引用, 可以赋值给Kotlin的非空类型, 不会有编译错误, 但是如果引用值为空, 会在运行时抛出异常.

举例, 在kotlin中调用Java的方法, 返回一个String:
```
val stringOne = JavaUtils.getStringOne()
println(stringOne)
println(stringOne.length)
```
如果Java方法返回null怎么办?
运行这段程序时先打出null, 再抛出`NullPointerException`.

如果程序是这样写的:
```
val stringOne: String = JavaUtils.getStringOne()
println(stringOne)
println(stringOne.length)
```
说明Kotlin假设Java传回来的是一个非空值. 这段代码编译时不会报错, 但是运行时第一行就抛出`IllegalStateException`.

如果这样写:
```
val stringOne: String? = JavaUtils.getStringOne()
println(stringOne)
println(stringOne?.length)
```
终于利用上了Kotlin的空安全检查, 所有用到这个变量的地方都要加上`?`, 如果不做检查编译时就会提示错误. 
但是这样防御难免导致代码太啰嗦了, 可能到处都是`?`.

好的实践是Java中的Public APIs(Non-primitive parameters, Field type, Return)都应该加上注解.

如果Java的类型上有关于null的注解, 就会直接表示为Kotlin中为不为null或者可为null的对应类型.

注解可以来自于各种包中, 比如JetBrains提供的: `@Nullable`和`@NotNull`.

比如:
```
@NotNull
public static String getStringOne() {
    return "hello";
}
```
这样Kotlin代码就知道传过来的肯定是个非空值, 可以放心使用.

如果是`@Nullable`, 编译器就会提示使用前做检查.


### 转义在Kotlin中作为关键字的Java标识符
Kotlin中的关键字, 比如:
```
fun, in, is, object, typealias, typeof, val, var, when
```

如果Java代码中用了这些关键字, 在Kotlin中调用该Java代码就要用`进行转义.

比如如果java中有一个名称为`is`的方法, 在kotlin中想要调用:
```
foo.`is`(bar)
```

但是, 首先需要考虑是不是名字起得不好, 如果可以改名(不是第三方代码), 优先考虑改名.

比较常见的一个使用情形是在写测试的时候, Mockito中的`when`就需要转义:
```
Mockito.`when`(xxx.foo()).thenReturn(yyy)
```
因为Mockito是一个Java的第三方库, 我们没法改它.

另一个解决办法是使用import alias, 给这个方法取个别名:
```
import org.mockito.Mockito.`when` as whenever
```
这样在使用的时候就可以用`whenever`来代替了`when`了.
import alias通常用来解决命名冲突的问题.

### SAM Conversions
SAM: Single Abstract Method.

只要函数参数匹配, Kotlin的函数可以自动转换为Java的接口实现.

Convention: 可以做SAM转换的参数类型应该放在方法的最后, 这样看起来更舒服.

举例, 如果在Java中定义方法:
```
interface Operation {
    int doCalculate(int left, int right);
}

public static int calculate(Operation operation, int firstNumber, int secondNumber) {
    return operation.doCalculate(firstNumber, secondNumber);
}
```
在Kotlin中调用的时候用SAM转换, 用一个lambda作为接口实现:
```
JavaUtils.calculate({ number1, number2 -> number1 + number2 }, 2, 3)
```
这样虽然正确, 但是可以改进.
把Java方法定义中的参数位置交换一下, 把接口参数放在最后:
```
public static int calculate(int firstNumber, int secondNumber, Operation operation) {
    return operation.doCalculate(firstNumber, secondNumber);
}
```
在Kotlin中, 最后一个lambda参数可以提取到括号外面:
```
JavaUtils.calculate(2, 3) { number1, number2 -> number1 + number2 }
```
这样看起来更好.

注意: SAM conversion只应用于java interop. 

上面的例子, 如果接口和方法是在Kotlin中定义的:
```
interface Operation2 {
    fun doCalculate(left: Int, right: Int): Int
}

fun calculate2(firstNumber: Int, secondNumber: Int, operation: Operation2): Int {
    return operation.doCalculate(firstNumber, secondNumber)
}
```
SAM conversions就不能用了, IDE会提示无法识别.
调用这个方法时, 第三个参数必须写成这种(匿名类的对象, 实现了接口):
```
calculate2(2, 3, object : Operation2 {
    override fun doCalculate(left: Int, right: Int): Int {
        return left + right
    }
})
```
这是因为在Kotlin的世界里, 函数是第一公民.

如果把前面的方法参数改为function type:
```
fun calculate3(firstNumber: Int, secondNumber: Int, operation: (Int, Int) -> Int): Int {
    return operation.invoke(firstNumber, secondNumber)
}
```
就可以像之前SAM conversions似的使用:
```
calculate3(2, 3) { number1, number2 -> number1 + number2 }
```

如果接口是在Java中定义, 但是接收参数的方法是Kotlin的方法:
```
fun calculate4(firstNumber: Int, secondNumber: Int, operation: JavaUtils.Operation): Int {
    return operation.doCalculate(firstNumber, secondNumber)
}
```
仍然是不能用SAM conversions, 因为这个方法仍然是可以接受函数类型的参数的.
在Kotlin中调用:
```
calculate4(2, 3, object : JavaUtils.Operation {
    override fun doCalculate(left: Int, right: Int): Int {
        return left + right
    }
})
```
IDE会提示你简化为:
```
calculate4(2, 3, JavaUtils.Operation { left, right -> left + right })
```
注意这里接口名称不能省略.

是不是感觉有点晕, 我把上面提到的几个调用情况写在一起:
```
// java function, java interface parameter
private fun trySAM1() {
    JavaUtils.calculate(2, 3) { number1, number2 -> number1 + number2 }
}

// kotlin function, kotlin interface parameter
private fun trySAM2() {
    calculate2(2, 3, object : Operation2 {
        override fun doCalculate(left: Int, right: Int): Int {
            return left + right
        }
    })
}

// kotlin function, function type parameter
private fun trySAM3() {
    calculate3(2, 3) { number1, number2 -> number1 + number2 }
}

// kotlin function, java interface parameter
private fun trySAM4() {
    calculate4(2, 3, JavaUtils.Operation { left, right -> left + right })
}

```
可以互相比较一下, 看看区别.


### Getter和Setter
Java中的getter和setter在Kotlin中会表现为properties. 但是如果只有setter, 不会作为可见的property.

### 异常
Kotlin中所有的异常都是unchecked的, 所以如果调用的Java代码有受检异常, kotlin并不会强迫你处理.

### 其他
Java中返回void的方法在Kotlin中会变成`Unit`.

`java.lang.Object`会变成`Any`, `Any`中的很多方法都是扩展方法.

Java没有运算符重载, 但是Kotlin支持. (运算符重载容易存在过度使用的问题.)

## Java调用Kotlin
### 属性
Kotlin的属性会被编译成Java中的一个私有字段, 加上getter和setter方法.

如果想要作为一个字段, 可以加上`@JvmField`注解.

### 包级别的方法
如果在一个文件`app.kt`中定义方法, 包名是`org.example`, 会被编译成Java的静态方法, Java类的类名是`org.example.AppKt`.

应用场景举例: 旧代码中有一个Java的辅助类, 包含静态方法:
```
public class Utils {
    public static int distanceBetween(int point1, int point2) {
        return point2 - point1;
    }
}
```

要把这个辅助类迁移到Kotlin代码, 可以新建一个Kotlin文件`DistanceUtils.kt`, 直接写包级别的方法:
```
fun distanceBetween(point1: Int, point2: Int): Int {
    return point2 - point1
}
```
在Java中调用这个方法的时候:
```
DistanceUtilsKt.distanceBetween(7, 9);
```
如果原先的Java代码中包含调用这个方法的地方太多, 又不想改所有的usage, 怎么办? -> 
可以通过注解`@file:JvmmName("xxx")`改变类名. 这样原先Java代码中调用的地方就避免了修改.

如果有两个文件指定了相同的JvmName, 编译会报错. 可以通过加上`@file:JvmMultifileClass`来解决. 这样多个Kotlin文件中定义的辅助方法对于Java来说会统一到同一个类中.

### 实例字段
如你需要把kotlin的property作为字段暴露出来, 可以加上`@JvmField`注解. 

适用的property: 有backing field, 没有这些修饰符: `private`, `open`, `override`, `const`, 也不是代理属性.

lateinit的属性会自动暴露为fields, 可见性和属性的setter一致.

Kotlin的data class会自动生成getter/setter. 如果加上`@JvmField`, 会直接暴露这些字段.

可以通过`@get:JvmName("xxx")`和`@set:JvmName("xxx")`来定制getter和setter的名字.


### 静态字段
Kotlin在有名字的object或者companion object中声明的属性, 将会编译成静态字段.

通常这些字段是private的, 不过也可以通过以下几种方式暴露:
* `@JvmField`.
* `lateinit`.
* `const`.

### 静态方法
前面提过, Kotlin包级别的方法会被编译成静态方法. 

在object或companion object中声明的方法, 默认是类中的实例方法. 比如:
```
class StaticMethodsDemoClass {
    companion object {
        fun sayHello() {
            println("hello")
        }
    }
}

object SingletonObject {
    fun sayWorld() {
        println("world")
    }
}
```
在Java中调用的时候:
```
StaticMethodsDemoClass.Companion.sayHello();
SingletonObject.INSTANCE.sayWorld();
```

如果给object或companion object中的方法加上`@JvmStatic`, 会生成一个静态方法和一个实例方法.
调用的时候就可以省略掉中间的`Companion`或`INSTANCE`关键字, 以类名直接调用静态方法.

比如:
```
class StaticMethodsDemoClass {
    companion object {
        fun sayHello() {
            println("hello")
        }

        @JvmStatic
        fun sayHelloStatic() {
            println("hello")
        }
    }
}
```
调用的时候:
```
StaticMethodsDemoClass.Companion.sayHello();
//StaticMethodsDemoClass.sayHello(); // error

StaticMethodsDemoClass.Companion.sayHelloStatic(); // ok, but not necessary
StaticMethodsDemoClass.sayHelloStatic();
```

`@JvmStatic`也可以用于属性, 就会有静态版本的getter和setter方法.

### 其他
Kotlin方法支持默认参数, 在Java中只有全部参数的方法签名才是可见的. 如果你希望对Java暴露多个方法重载, 要给方法加上`@JvmOverloads`.
比如在Kotlin中写一个自定义View的构造函数:
```
class DialView @JvmOverloads constructor(
   context: Context,
   attrs: AttributeSet? = null,
   defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
}
```

还可以利用
`@JvmName`来给方法重命名. 因为在Kotlin中是扩展方法, 在Java中只是一个静态方法, 名字可能不够直观.


Kotlin没有checked exceptions.
如果想在Java中调用一个Kotlin方法, 并包一个try-catch, 会报错说没有抛出这个异常.

可以在Kotlin方法中加上注解, 比如`@Throws(IOException::class)`.

## Feature leak prevention
Kotlin方法的参数名, 在生成代码中会作为一个字符串出现, 从而不会被混淆, 有可能会泄漏. 所以不建议放敏感信息到参数名中.

类似的还有字段名, 扩展方法名.

可以在proguard中加上
```
-assumenosideeffects class kotlin.jvm.internal.Intrinsics {
    public static void checkParameterIsNotNull(...);
    public static void throwUninitializedPropertyAccessException(...);
}
```
来移除这些代码.
但是建议在测试环境中仍然保留这些代码, 以便有错误发生的时候能够快速发现.


## Tools
### IDE的自动转换
`Code -> Convert Java File to Kotlin File`.

如果粘贴Java代码到.kt文件, IDE会自动将所粘贴代码转换为Kotlin代码.

### 查看编译成的Java代码
在IDE里面可以显示Kotlin Bytecode, 然后decompile, 显示java代码.

`Tools -> Kotlin -> Show Kotlin Bytecode -> Decompile`.


## 参考
* [官方文档: Calling Java code from Kotlin](https://kotlinlang.org/docs/reference/java-interop.html)
* [官方文档: Calling Kotlin from Java](https://kotlinlang.org/docs/reference/java-to-kotlin-interop.html)
* [Codelab: Refactoring to Kotlin](https://codelabs.developers.google.com/codelabs/java-to-kotlin/#0)
* [Java ❤️ Kotlin, Happy Together 🎵 (Android Dev Summit '19)](https://www.youtube.com/watch?v=LZFzRXCO95o&list=PLWz5rJ2EKKc_xXXubDti2eRnIKU0p7wHd&index=9)