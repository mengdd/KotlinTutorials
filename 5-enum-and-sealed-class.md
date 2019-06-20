# 枚举和Sealed Class
## 枚举
首先, Kotlin和Java一样, 也是有枚举类型的:
```kotlin
enum class Direction {
    NORTH, SOUTH, WEST, EAST
}

enum class Color(val rgb: Int) {
    RED(0xFF0000),
    GREEN(0x00FF00),
    BLUE(0x0000FF)
}
```
枚举类型还可以实现接口(但是不能继承类), 提供一样或者不一样的成员实现.

和Java一样, 枚举类型也有`valueOf`和`values()`方法:
```kotlin
EnumClass.valueOf(value: String): EnumClass
EnumClass.values(): Array<EnumClass>
```
## Sealed Class
枚举是Java的同类产物, 而sealed class则是kotlin推出的新产品.
(C#中也有: [sealed class in C sharp](https://www.c-sharpcorner.com/article/sealed-class-in-C-Sharp/))

首先定义一个sealed class, 它是抽象的, 是用来被继承的, 但是它又限制了继承的自由, 它的子类就是有限的几种情况.

### 怎么写sealed class:

```kotlin
sealed class Expr
data class Const(val number: Double) : Expr()
data class Sum(val e1: Expr, val e2: Expr) : Expr()
object NotANumber : Expr()
```
其中`Const`和`Sum`可以被继承, `NotANumber`实际上是一个单例.

* sealed class所有的子类必须和sealed class在同一个文件声明. (注意如果是子类的子类(间接继承), 则可以在任何地方.)
* sealed class本身是抽象的, 不能被直接实例化, 可以有`abstract`成员.
* sealed class的构造是私有的, 不允许有非私有构造.


### 应用场景
sealed class常用在when表达式中.
如果所有情形都覆盖到了, 可以省略else.
```kotlin
fun eval(expr: Expr): Double = when (expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    NotANumber -> Double.NaN
    // the `else` clause is not required because we've covered all the cases
}
```
用when表达式时, 如果有分支没有被覆盖到, 并且没有提供else, 编译会有错误提示的.

### sealed class和enum class的比较
sealed class用来表达有限的继承体系, 是枚举类型的一种扩展形式.

区别:
* 枚举类型的值是有限的, 每个枚举常量仅作为一个单例存在.
* sealed class的子类可以有多个实例, 并且每个实例包含自己的状态.

Sealed Class比枚举更方便的地方:
* 枚举要求构造相同, 但sealed class可以传入不同的实例域.
比如上面例子中的`Const`和`Sum`, 它们的构造传入的参数就不同.

## 参考
* [Enum Classes](https://kotlinlang.org/docs/reference/enum-classes.html)
* [Sealed Classes](https://kotlinlang.org/docs/reference/sealed-classes.html)
* [Kotlin Sealed Classes — enums with swag](https://proandroiddev.com/kotlin-sealed-classes-enums-with-swag-d3c4b799bcd4)