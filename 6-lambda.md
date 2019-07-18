# Lambda表达式和高阶函数
在Kotlin中函数是第一公民, 不像Java一样, 函数一定需要写在某个类里面, Kotlin中的函数也可以直接写在文件里.

Lambda表达式并不是什么新东西, Java 8就有了.
它的存在主要是为了让代码更加简洁.

高阶函数呢, 基本概念也很简单, 就是函数, 同样也可以像其他类型的对象一样, 作为一个函数的参数和返回值.

## Lambda表达式
lambda表达式和匿名函数都是函数字面值(function literals), 它们没有提前声明, 直接作为表达式传入.

### Lambda表达式的语法
lambda表达式的构成: 
```
{ 参数列表 -> 函数体 }
```
其中参数类型是可以省略的.
如果函数体的返回值不是`Unit`, 那么最后一个表达式会被当做返回值.

### 举例: setOnClickListener
对Lambda表达式的应用很大程度上要感谢现代化的IDE, 比如一个简单的给button设置listener的代码, 

最开始, 作为一个把Kotlin当Java 7用的人, 可能会写成这样:
```kotlin
button.setOnClickListener(object : View.OnClickListener{
    override fun onClick(v: View?) {
    }
})
```
这里利用了`object`关键字, 声明了一个匿名类的对象.

但是IDE看到这个代码就不那么开心了, 出现了一条黄色的下划线, Alt + Enter, "Convert to lambda", 再Enter确认, 就变成了这样:
```kotlin
button.setOnClickListener { }
```
为什么可以这样干呢, 这是因为`View`的`OnClickListener`接口是一种特殊类型的接口: 只有一个方法.

这种只拥有一个方法的接口被称作是**函数式接口**(functional interface), 也被叫做单抽象方法类型, 即`SAM`, (single abstract method).

PS: 一个错误的示范:
如果你不幸一开始把代码写成了这样:
```kotlin
button.setOnClickListener{object : View.OnClickListener {
    override fun onClick(v: View?) {
    }
}}
```
注意这里与上面例子的区别仅仅是把小括号()变成了大括号{}.
程序没有报错, 但按钮点击的时候应该是执行不到你想要的代码了.

IDE仍然会提示你简化, 简化后变成了这样:
```kotlin
button.setOnClickListener{ View.OnClickListener { } }
```
也就是说每次按钮点击, 你做的事情都是创建了一个匿名对象.

为什么会犯这种错误, 而IDE又不提示呢, 这是因为lambda的惯例允许这样的写法, 这么写虽然逻辑上有问题, 但是语法上是没有错误的. 
下面请看都有什么惯例呢.

### lambda惯例
* 如果函数的最后一个参数接收函数, 传入的lambda表达式可以写在圆括号外面. (这种语法称为trailing lambda).
在这种情况下, 如果lambda是唯一的参数, 那么圆括号可以省略. (上面的错误例子就是因为符合了这个惯例, 所以语法上没有错误).
* 如果lambda只有一个参数, 可以不声明直接用, 隐式名字`it`.
* lambda表达式的返回值: 可以显式`return`, 如果不显式返回, 默认返回最后一个表达式的值.
* 没有用到的参数可以用下划线`_`表示.

### 匿名函数
lambda表达式并不能显式指定返回类型, 大多数情况下可能用不上这一点, 因为往往返回类型都是可以被自动推断出来的.

但是如果你真的需要显式指定, 你可以用匿名函数.
```kotlin
fun(x: Int, y: Int): Int = x + y
```
匿名函数看起来和普通的函数声明很像, 只是它没有名字.

和普通函数不同的是, 如果参数类型可以从上下文推断出来, 那么可以省略不写参数类型:
```
ints.filter(fun(item) = item > 0)
```

匿名函数的返回类型推断和正常函数一样.

**匿名函数与Lambda的区别**:
* 匿名函数的参数永远是在圆括号内传递的, 把函数参数放在圆括号外的简化语法只适用于lambda表达式.
* 不带标签的return语句: 在lambda中, 返回外层的函数, 在匿名函数中, 返回本函数.


## 高阶函数
高阶函数(Higher-Order Functions): 把函数作为参数或返回值的函数.

## 函数类型
函数类型(Function types): 函数根据参数和返回值, 可以归类到一个函数类型. 

### 函数类型的写法
函数类型的基本写法: 括号中的参数列表和一个返回值. 比如`(A, B) -> C`就是一个函数类型, 这种类型的函数接受A和B两种类型的参数, 返回一个类型C的返回值. 
* 参数列表可以为空, 比如`() -> A`. 
* 返回值为`Unit`时不能省略.
* 函数类型还可以写接收器(receiver)类型: `A.(B) -> C`. 表示这种函数是在A类型上调用的.
* `suspend`关键字表示一种特殊的函数类型, 所以如果有, `suspend`修饰符也要出现. 比如`suspend () -> Unit`.
* 也可以加上参数名, 来表达参数的意义. 如`(x: Int, y: Int) -> Point`.

几点比较特殊的:
* 函数类型也有可空类型, 加括号和`?`. 比如`((Int, Int) -> Int)?`.
* 函数类型可以是高阶的, 用括号组合, 如: `(Int) -> ((Int) -> Unit)`.
* 箭头是按照右结合的原则, 即运算按照从右向左的顺序, 所以`(Int) -> (Int) -> Unit`和`(Int) -> ((Int) -> Unit)`是一个意思.

可以给函数类型一个类型别名, 比如:
```kotlin
typealias ClickHandler = (Button, ClickEvent) -> Unit
```
### 实例化函数类型
实例化函数类型有好几种方法:
* 用函数字面值(function literal)的代码块: lambda表达式, 匿名函数.
* 使用已有的声明引用: 函数, 属性, 构造函数. `::`的用法参见[Callable Reference](https://kotlinlang.org/docs/reference/reflection.html#callable-references).
* 函数类型还可以当做接口被类实现, 那么创建这个类的实例就是创建了函数类型的实例.

### 调用函数类型的实例
函数类型声明了, 也实例化了, 怎么调用呢?
可以用`invoke`操作符, 也可以直接用名称调用.

如果有接受者类型(receiver), 那么接受者对象需要作为第一个参数.
另一种方式也可以将接受者对象放在点(`.`)前面, 像扩展函数一样调用.

```kotlin
val stringPlus: (String, String) -> String = String::plus
val intPlus: Int.(Int) -> Int = Int::plus

println(stringPlus.invoke("<-", "->"))
println(stringPlus("Hello, ", "world!")) 

println(intPlus.invoke(1, 1))
println(intPlus(1, 2))
println(2.intPlus(3)) // extension-like call
```

## 参考
* [Higher-Order Functions and Lambdas](https://kotlinlang.org/docs/reference/lambdas.html)
* [Functions](https://kotlinlang.org/docs/reference/functions.html)
* [Returns and Jumps](https://kotlinlang.org/docs/reference/returns.html)
* [Callable Reference](https://kotlinlang.org/docs/reference/reflection.html#callable-references)