# 给Java开发者看的Kotlin基础
Java开发者看完这篇基础(再加上后面一篇空安全相关)就可以开始上手写代码了.
后面具体的点可以边用边学.

## 和Java的基本区别
* 没有`new`关键字, 直接创建对象.
* 没有`;`.
* 类型分为Nullable(带`?`)和Non-Null.
* 继承类和实现接口都用`:`.

## 类型
在Kotlin中支持的数据类型:
`Double`, `Float`, `Long`, `Int`, `Short`, `Byte`.
不再像Java一样支持小数到大数的默认转换, 所有的转换都要通过显式的方法, 比如`toInt()`方法.

字符类型: `Char`, 不能像Java中一样直接当数字对待了.
布尔类型: `Boolean`.

以上类型在运行时都会被表示为JVM原生类型(除非是可空类型和泛型应用), 但是在代码中它们只有一种写法(不像Java中有int和Integer两种), 
在用户看来它们就是普通的类.

所有的类型需要可空(nullable)类型时, 会被自动装箱.

字符串类型: `String`: 字符串是不可变(immutable)类型, 可以用`[]`访问元素, 可以用`$`加入模板表达式(可以是变量也可以是用`{}`包含的表达式).


## 方法声明
方法声明格式:
```
fun 方法名(参数): 返回值类型 {
    方法体
}
```
参数列表中变量名在前面, 变量类型在后面.

比如:
```
fun sum(a: Int, b: Int): Int {
    return a + b
}
```
方法体要是返回一个简单的表达式, 可以直接用`=`:
```
fun sum(a: Int, b: Int) = a + b
```
这种情况下由于返回值类型可以被推断出来, 所以也可以省略不写.

返回值为空: `Unit`可以省略不写.

当方法有body时, 除非返回值是`Unit`, 否则返回值类型是不能被省略的.

### 方法可以有默认参数
声明方法的时候, 可以用`=`给参数设置默认值.
当调用方法时省略了该参数, 就会使用默认值.

调用方法的时候可以指定参数的名字.
```
fun reformat(str: String,
             normalizeCase: Boolean = true,
             upperCaseFirstLetter: Boolean = true,
             divideByCamelHumps: Boolean = false,
             wordSeparator: Char = ' ') {
...
}
```
调用的时候:
```
reformat(str, wordSeparator = '_')
```

文档: [Functions](http://kotlinlang.org/docs/reference/functions.html)

## 变量声明
变量声明有两个关键字: `val`和`var`.

其中`val`只能被赋值一次, 相当于final.

```
val a: Int = 1  // immediate assignment
val b = 2   // `Int` type is inferred
val c: Int  // Type required when no initializer is provided
c = 3       // deferred assignment
```
变量声明时如果初始化可以推断出类型, 则可以省略类型.

## 流程控制
### if
在kotlin中的if是一个表达式(expression), 即它可以返回值.

它的分支可以是{}包围的block, 其中最后一个表达式就是对应分支的值.
```
val max = if (a > b) {
    print("Choose a")
    a
} else {
    print("Choose b")
    b
}
```
当if作为表达式被使用的时候, 它必须有else.

Java中常用的三元运算符`条件? 满足 : 不满足`, 在Kotlin中是不存在的.

Kotlin中的Elvis Operator是用来进行null判断然后选择处理的. (具体见下篇空安全文章介绍)


### when
when是用来取代Java中的switch的.

```
when (x) {
    1 -> print("x == 1")
    2 -> print("x == 2")
    else -> { // Note the block
        print("x is neither 1 nor 2")
    }
}
```

和if一样, when也是可以被用作expression(表达式)或statement. 作为表达式使用的时候, else是必须的(除非编译器可以推断所有的情况都已经被覆盖了).

### 循环
`while`, `do...while`, `break`, `continue`的用法都和Java一样.

`for`的用法也类似, 只不过在Java中是`:`, 在Kotlin中变成了`in`.

Kotlin有一些Range操作符: [Ranges](http://kotlinlang.org/docs/reference/ranges.html)

## 参考资料
* [Kotlin Basic Syntax](http://kotlinlang.org/docs/reference/basic-syntax.html)
* [Basic Types](https://kotlinlang.org/docs/reference/basic-types.html)
* [Functions](https://kotlinlang.org/docs/reference/functions.html)
* [Control Flow](https://kotlinlang.org/docs/reference/control-flow.html)
