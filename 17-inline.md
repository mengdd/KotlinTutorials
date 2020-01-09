# Kotlin中的inline, noinline, crossinline, reified
* Kotlin中的`inline`, `noinline`, `crossinline`都是什么意思? 干什么用的?
* Kotlin中的`reified`又是干什么用的?

本篇文章介绍Kotlin的inline函数, 顺一顺相关的知识点, 解决这些问题.

## inline: 内联
最开始接触`inline`这个词是学C/C++的时候, 叫内联. 编译器会把函数体替换在函数被调用的地方. 

Kotlin中的`inline`也是这个意思, 主要是解决了函数调用时的开销, 调用栈的保存, 匿名对象的建立等. 因为Kotlin支持高阶函数, lambda等, 所以用`inline`帮助降低一些运行时开销.

`inline`让编译器直接把代码复制到调用的地方, 比起直接复制粘贴代码, 同时又保持了函数的复用性和可读性.

Java在语言层面暂时不支持inline, JVM会做一些相关的优化.

### inline做什么
举个例子来看看inline和不inline的代码有什么区别:

```
fun main() {
    sayHi {
        println("I'm wind, what's your name?")
    }
}

fun sayHi(body: () -> Unit) {
    println("Hi, ")
    body()
    println("Bye!")
}
```

decompile后的Java代码:
```
public static final void main() {
  sayHi((Function0)null.INSTANCE);
}

public static final void sayHi(@NotNull Function0 body) {
  Intrinsics.checkParameterIsNotNull(body, "body");
  String var1 = "Hi, ";
  boolean var2 = false;
  System.out.println(var1);
  body.invoke();
  var1 = "Bye!";
  var2 = false;
  System.out.println(var1);
}
```
`main`调用了`sayHi`, `sayHi`里面又执行了`body`.


如果只改动一行, 给`sayHi`方法加上`inline`关键字:
```
inline fun sayHi(body: () -> Unit) {
    println("Hi, ")
    body()
    println("Bye!")
}
```
那么decompile后:
```
public static final void main() {
	int $i$f$sayHi = false;
	String var1 = "Hi, ";
	boolean var2 = false;
	System.out.println(var1);
	int var3 = false;
	String var4 = "I'm wind, what's your name?";
	boolean var5 = false;
	System.out.println(var4);
	var1 = "Bye!";
	var2 = false;
	System.out.println(var1);
}
```
可以看到inline之后, `main`中的代码就是实际做事情的代码, 它不知道自己调用了`sayHi`, 也没有为lambda参数`body`建立对象.

如果这个方法是在循环中调用的, 加个`inline`关键字可以省下不少对象的建立.

`inline`修饰符同时作用于函数本身和它的函数类型参数: 它们都被inline到被调用的地方了.


### 什么时候使用inline
如果你在一个很简单的非高阶函数前面加上`inline`,
举个例子:
```
inline fun sayName() {
    println("Wind")
}
```
那么你会遇到IDE把`inline`标黄, 并且提示:
```
Expected performance impact from inlining is insignificant. Inlining works best for functions with parameters of functional types.
```
因为这样做没有什么必要了.

inline的适用场景: 高阶工具类函数.
比如`filter`, `map`, `joinToString`, `repeat`等.

`inline`不适用于: 
* 很大很长的函数.


## noinline
`noinline`又是用来干啥呢?

前面说过`inline`同时作用于函数本身和它的函数(lambda)参数. 如果函数有多个函数参数, 有些我不希望被inline, 那就可以用`noinline`来修饰.
```
inline fun aMixedInlineFunction(inlined: () -> Unit, noinline notInlined: () -> Unit) {
    inlined()
    notInlined()
}
```

## crossinline
还是按上面的思路, 先看看`crossinline`出现的原因.

首先复习一下`return`的相关知识点.

### local return
* 一般情况下, 方法里面的lambda是不能return外部函数的.

举例: 这是个普通的高阶方法, 带有一个lambda参数:
```
fun fooNormal(body: () -> Unit) {
    println("normal start")
    body()
    println("normal done")
}
```
它被调用的时候, 如果想在lambda中直接return:
```
fun main() {
    fooNormal {
        println("body 1")
        return // return is not allowed here
        return@fooNormal // return@fooNormal is allowed
    }
}
```
return会被标红, 提示`return is not allowed here`.

只能带上一个label写`return@fooNormal`, 表示只是退出当前这个lambda, 而不是退出外面的函数.

这种叫做`local return`, 因为只退出了最近的闭包.
lambda闭包之外, 函数后面的语句还是会照常执行.

### non-local return
* `inline`方法里面的lambda可以return外部函数. 

把上面的例子稍微改一下, 把方法改成inline的:
```
inline fun fooInline(body: () -> Unit) {
    println("inline start")
    body()
    println("inline done")
}
```
调用的时候:
```
fun main() {
    fooInline {
        println("body 2")
        return
    }

    println("the end of main")
}
```
这时候就可以在lambda里面直接写`return`了.


运行结果:
```
inline start
body 2
```
可以看到不仅`fooInline`方法后面的语句没有被执行, 连`main`都退出了. 联想一下`inline`的原理, 很好理解.

在这种情况下, lambda中的return实际上是作用于方法的调用处的. 这就是著名的`non-local return`.

很多集合的方法都是`inline`的,

这就是为什么在`forEach`中可以直接用`return`从方法中跳出来:
```
fun hasZeros(ints: List<Int>): Boolean {
    ints.forEach {
        if (it == 0) return true // returns from hasZeros
    }
    return false
}
```
### crossinline : disable non-local return
但是有时候作为参数传入的lambda不一定是被函数直接使用, 有可能会被嵌套. 

在这种情况下, 规范干脆规定禁止了non-local return, 否则容易写出混乱的代码.

比如这个方法:
```
inline fun fooWithCrossinline2(body: () -> Unit) {
    val f = Runnable { body() } // Error
    println("fooWithCrossinline 2")
}
```
这样写直接就报错了:
```
Can't inline `body` here: it may contain non-local returns. Add `crossinline` modifier to parameter declaration `body`
```
这个提示明明白白, 此时按下`Alt+Enter`, 给参数加上`crossinline`即修好:
```
inline fun fooWithCrossinline2(crossinline body: () -> Unit) {
    val f = Runnable { body() }
    println("fooWithCrossinline 2")
}
```

调用这个方法的时候, 如果在lambda中企图进行non-local return, 会和普通方法一样提示不行:
```
fooWithCrossinline2 { 
    return // Error: return is not allowed here
}
```

即便内部使用没有什么嵌套关系, 如果函数的设计者想禁止non-local return, 也是可以直接将参数标记为`crossinline`的.
```
inline fun fooWithCrossinline(crossinline body: () -> Unit) {
    println("with crossinline start")
    body()
    println("with crossinline done")
}
```
使用的时候, 如果企图non-local return也是同样报错:
```
fooWithCrossinline {
    return // return is not allowed here
}
```


对`crossinline`总结一下:
* 我怎么知道某个`inline`函数的某个lambda参数在内部使用时到底有没有嵌套关系? -> 如果有嵌套, 它必定被标记为`crossinline`, 必定不能non-local return.
* 虽然没有嵌套关系, 但是想禁止在lambda中直接return外部函数 -> 把参数标记为`crossinline`.


## reified
有时候我们需要类型作为参数, 但是又觉得函数声明个`clazz: Class<T>`参数, 传入实参`MyClass::class.java`这样比较难看.

我这么说可能不太好明白, 还是举个例子吧.

比如这是一个查找某个类型实例的查找方法:
```
interface Hero
class SuperMan : Hero
class Hulk : Hero
class IronMan : Hero

fun <T> findHero1(candidates: List<Hero>, clazz: Class<T>): T? {
    candidates.forEach {
        if (clazz.isInstance(it)) {
            @Suppress("UNCHECKED_CAST")
            return it as T
        }
    }
    return null
}
```
调用这个方法的时候, 参数是这么传的:
```
findHero1(candidates, Hulk::class.java)
```

能不能就只传入类名呢?

既然这么问了当然是可以的.
inline函数支持reified type parameters, 可以写成这样:
```
inline fun <reified T> findHero2(candidates: List<Hero>): T? {
    candidates.forEach {
        if (it is T) {
            return it as T
        }
    }
    return null
}
```
此时调用查找方法:
```
findHero2<SuperMan>(candidates)
```

用了`reified`之后, T可以直接当做类型来使用了, 并且不再需要反射, `is`和`as`等操作符都可以用了. 也去掉了那个丑陋的`@Suppress`.

注意:
* 只有`inline`函数的参数可以被标记为`reified`.
* 只有runtime-available的类型可以被传入`reified`类型的参数. Nothing, List<T>不行.

## 访问限制
因为函数默认是public的, 当一个方法inline之后, 它就作为public API了, 不能访问私有字段.

把字段改为`internal`并且加上注解`@PublishedApi`之后可以访问:


```
class PublishedApiDemo {
    @PublishedApi
    internal var internalField = "internal published api"
    private var somePrivateField = "private field"

    inline fun someInlineFun(body: () -> Unit) {
        //somePrivateField.length //ERROR
        body()
        internalField //OK
    }
}

```

## Recap
* `inline`解决函数调用开销.
* `noinline`阻止参数被inline.
* `crossinline`阻止non-local return.
* `reified`让类型参数更加具体, 好用.

## 参考
* [Inline Functions](https://kotlinlang.org/docs/reference/inline-functions.html)
* [Reified Type Parameters](https://github.com/JetBrains/kotlin/blob/master/spec-docs/reified-type-parameters.md)
* [Effective Kotlin: Consider inline modifier for higher-order functions](https://blog.kotlin-academy.com/effective-kotlin-consider-inline-modifier-for-higher-order-functions-758afcaffc11)
* [Inline Functions in Kotlin](https://www.baeldung.com/kotlin-inline-functions)