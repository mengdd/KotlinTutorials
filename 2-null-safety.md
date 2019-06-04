# 空安全: Kotlin的一大卖点
Kotlin的type system旨在减少`NullPointerException`(`NPE`). 

主要是通过编译期间, 就区分了哪些东西是不会为null的, 哪些是可能为null的. 
如果想要把null传递给不能为null的类型, 会有编译错误.
可能为null的引用使用时必须要做相应的检查.

```
var a: String = "abc"
// a = null // compile error
val aLength = a.length

var b: String? = "abc"
b = null // ok
// val bLength = b.length // 不能直接使用, compile error
```

## 检查是否为null
对于可能为null的类型, 可以用if来做显式的检查.
但是这种只适用于变量不可变的情形, 免得刚检查完是否为null, 它就变了.

## Safe Calls
可以用操作符`?.`来进行安全操作.
```
var b: String? = "abc"
val bLength = b?.length // 如果b不为null, 返回长度, 如果b为null, 返回null
```
上面bLength类型为`Int?`.

Safe calls在链式操作时非常有用.
比如:
```
bob?.department?.head?.name
```
其中任何一个环节为null了最后的结果就为null.


还可以用在表达式的左边:
```
// If either `person` or `person.department` is null, the function is not called:
person?.department?.head = managersPool.getManager()
```

### `let()`
实践中比较常见的是用`let()`操作符, 对非null的值做一些操作:
```
val listWithNulls: List<String?> = listOf("Kotlin", null)
for (item in listWithNulls) {
    item?.let { println(it) } // prints A and ignores null
}
```

## Elvis Operator
我们可能有这样的需求, 有个可能为null的变量, 我们需要在其不为null的时候返回一个表达式, 为null的时候返回一个特定的值.

比如:
```
val l: Int = if (b != null) b.length else -1
```
我们可以写成这样:
```
val l = b?.length ?: -1
```
仅当`?:`左边为null的时候, 右边的表达式才会执行.

实际应用:
在Kotlin中, return和throw都是表达式, 所以我们可以这样用:
```
fun foo(node: Node): String? {
    val parent = node.getParent() ?: return null
    val name = node.getName() ?: throw IllegalArgumentException("name expected")
    // ...
}
```

## 操作符!!
`!!`操作符表达的是: "嘿, 肯定不为null".
所以它称作not-null assertion operator.
它的作用是把原来可能为null的类型强转成不能为null的类型.

如果这种断言失败了, 就抛出NPE了.


## 仍然可能会遇到NPE的情形
* `throw NullPointerException()`.
* `!!`使用在了为null的对象上.
* 初始化相关的一些数据不一致情形.
    - 一个构造中没有初始化的`this`传递到其他地方使用.
    - 基类的构造中使用了open的成员, 实现在子类中, 此时可能还没有被初始化.
* 和Java互相调用的时候:
    - Java声明的类型叫[platform type](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types), 其null安全和Java中的一样.
    - 和Java互相调用时的泛型使用了错误的类型. 比如Kotlin中的`MutableList<String>`, 在Java代码中可能加入一个null. 此时应该声明为`MutableList<String?>`.
    - 其他外部Java代码导致的情形.

## 相关应用
### 安全强转
强转时如果类型不匹配会抛出`ClassCastException`
可以用`as?`来做安全强转, 当类型不匹配的时候返回null, 而不是抛出异常.

```
val aInt: Int? = a as? Int
```

### 集合过滤
集合中有个`filterNotNull()`可以用来过滤出非空元素.
```
val nullableList: List<Int?> = listOf(1, 2, null, 4)
val intList: List<Int> = nullableList.filterNotNull()
```

## 参考资料
* [Null Safety](https://kotlinlang.org/docs/reference/null-safety.html)
* [Calling Java from Kotlin: Null-Safety and Platform Types](https://kotlinlang.org/docs/reference/java-interop.html#null-safety-and-platform-types)