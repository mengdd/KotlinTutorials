# Scope Functions: Kotlin中的作用域函数
Kotlin标准库提供了5个scope functions(作用域函数): `let`, `run`, `with`, `apply`, `also`.

作用域函数的目的是为了在对象的上下文中执行一段代码.
当你在一个对象上调用作用域方法, 提供一个lambda表达式, 会形成一个临时的scope, 在这个scope里, 访问该对象可以不用它的名字.

作用域方法没有引入什么新的技术能力, 它们只是简化了代码.

## 作用域函数的区别
作用域函数主要的不同有三点:
* 访问上下文对象的方式: `this`还是`it`.
* 返回值: 是上下文对象还是lambda结果.
* 是否是扩展函数.

### 上下文对象:
* 使用`this`的: `run`, `with`, `apply`. 对象作为lambda的receiver.
(`this`可以被省略).
* 使用`it`的: `let`和`also`. 对象作为lambda的argument. 也可以提供自定义的名字, 不提供的话, 默认是`it`.

举例:
```
val str: String? = "Hello"
// this
str?.run {
    println("run: The receiver string's length is $length")
//  println("run: The receiver string's length is ${this.length}") // does the same
}

// it
str?.let {
    println("let: The receiver string's length is ${it.length}")
}
```

### 返回值:
* `apply`和`also`返回上下文对象自己. (可以记忆为`a`开头的两个方法返回自己.)
* `let`, `run`和`with`返回lambda的结果.

### 是否是扩展函数:
* `with`不是扩展函数.
* `run`有扩展函数和非扩展函数两种形式.
* `let`, `also`, `apply`都是扩展函数.

## 使用建议
* `let`: 在非空的值上执行一段代码 -> `?.let`; 在一个较小作用域内引入局部变量.


比如有数据结构:
```
data class Person(val name: String, var pet: Pet? = null)
data class Pet(var name: String? = null, var age: Int? = null, var type: String? = null)
```
`let`的非空判断可以级联:
```
person?.pet?.name?.let { println("pet name is: $it") }
```
可以结合`?:`来设置为空时做什么:
```
person?.pet?.name?.let { println("pet name is: $it") } ?: println("there is no pet name")
```
这样不论是person还是pet还是name为空, 都会打印出"there is no pet name".

* `with`: 需要在上下文对象上运行代码, 不需要返回值; 对对象进行一组函数调用.
* `run`: 对象配置, 并计算一个结果返回.
* `apply`: 对象配置. 比如Android中常见的Fragment和Intent的创建和参数设置.
```
val bundle = Bundle().apply {
    putString("ARG_XXX", value)
}

MyFragment().apply {
    arguments = bundle
})
```
上面这段代码也可以直接嵌套起来, 但是有的convention可能不推荐, 因为会增加理解成本. 所以要看团队意见. 

```
Intent().apply {
    action = Intent.ACTION_VIEW
    data = Uri.parse("some-uri")
}
```

* `also`: 做一些不改变对象本身的额外操作, 比如打印log.

注意有些使用场景是重合的, 所以可以自己根据实际情况选用.

注意不要过度使用: 
* 可能会降低可读性; 
* 避免嵌套使用作用域函数; 
* 链式使用的时候要注意上下文对象的值.

作用域函数和`takeIf`, `takeUnless`结合使用很方便.

## 参考
* [官方文档](https://kotlinlang.org/docs/reference/scope-functions.html)
* [Learn by Examples](https://play.kotlinlang.org/byExample/06_scope_functions/01_let)
* [Mastering Kotlin standard functions: run, with, let, also and apply](https://medium.com/@elye.project/mastering-kotlin-standard-functions-run-with-let-also-and-apply-9cd334b0ef84)