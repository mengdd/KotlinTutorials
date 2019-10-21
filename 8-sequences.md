# Sequences
## Sequences是用来干什么的
Kotlin标准库还提供了另一种容器类型: sequences.

`Sequence`接口和`Iterable`接口很像, 都是提供了遍历操作. 

不同点在于, 对于集合的多步操作, Sequences提供了不同的做法.

* 多步处理时, 对于`Iterable`, 执行是急切的(eagerly), 每一个步骤都完成和返回一个中间集合(collection), 后续的操作再在这个集合上继续执行.
* sequences的多步处理是延迟的(lazily), 只有在整个链条的结果被请求的时候才会真正执行每一步的计算.

操作符的执行顺序同样也是不同的:
* `Iterable`会对集合的所有元素先完成第一个操作, 然后整体再往下走, 对所有元素进行下一个操作.
* `Sequence`则按照元素, 每个元素完成一系列操作, 接着是下一个元素.

我想了一个比喻: 双层循环嵌套, `Iterable`的外层遍历操作符, 内层遍历元素; 而`Sequence`的外层遍历元素, 内层遍历操作符.

一段为了辅助理解操作顺序的伪代码:
```
// Iterable
for (operation in operators) {
    for (element in collection) {
        //...
    }
}
// Sequence
for (element in collection) {
    for (operation in operators) {
        //...
    }
}
```

## 举例说明
下面通过实际的例子来说明.

例子背景: 有一堆书, 书的类型是:
```
data class Book(val name: String, val author: String)
```
其中包含书名和作者名.

现在我们想要在这堆书里, 筛选出某些作者的书, 符合条件的三本就可以了, 并输出书名.

用一般collection的实现:
```
fun findBookNamesOfAuthorsUsingCollection(authors: Set<String>): List<String> {
    return books
        .filter {
            println("filter: $it")
            it.author in authors
        }
        .map {
            println("map: $it")
            it.name
        }
        .take(3)
}
```
其中`books`是一个List.
上面进行的操作: 从书里先根据作者过滤, 然后`map`变换到书名, 最后取三个书名返回.

但是需要注意的是:
* 每个操作符都会创建一个新的数组并输出, 作为下一级操作的输入.
* 如果书的数量比较多, 最后却只取了3本, 那么前两步过滤和映射操作会造成浪费.

这一点从console输出结果上就可以看出来:
```
filter: Book(name=Brown Bear, Brown Bear, What Do You See?, author=Eric Carle)
filter: Book(name=1, 2, 3 to the Zoo, author=Eric Carle)
filter: Book(name=The Very Hungry Caterpillar, author=Eric Carle)
filter: Book(name=Pancakes, Pancakes!, author=Eric Carle)
filter: Book(name=The Tiny Seed, author=Eric Carle)
filter: Book(name=Do You Want to Be My Friend?, author=Eric Carle)
filter: Book(name=The Mixed-Up Chameleon, author=Eric Carle)
filter: Book(name=The Very Busy Spider, author=Eric Carle)
filter: Book(name=Papa, Please Get the Moon for Me, author=Eric Carle)
filter: Book(name=Today Is Monday, author=Eric Carle)
filter: Book(name=From Head to Toe, author=Eric Carle)
filter: Book(name=Does A Kangaroo Have A Mother, Too?, author=Eric Carle)
filter: Book(name=10 Little Rubber Ducks, author=Eric Carle)
filter: Book(name=Where Is Baby's Belly Button?, author=Karen Katz)
filter: Book(name=No Biting!, author=Karen Katz)
filter: Book(name=I Can Share, author=Karen Katz)
filter: Book(name=My Car, author=Byron Barton)
filter: Book(name=My Bus, author=Byron Barton)
filter: Book(name=Dear Zoo, author=Rod Campbell)
filter: Book(name=Dr. Seuss's ABC, author=Dr. Seuss)
filter: Book(name=Fox in Socks, author=Dr. Seuss)
filter: Book(name=The Cat in the Hat, author=Dr. Seuss)
filter: Book(name=Hop on Pop, author=Dr. Seuss)
map: Book(name=Brown Bear, Brown Bear, What Do You See?, author=Eric Carle)
map: Book(name=1, 2, 3 to the Zoo, author=Eric Carle)
map: Book(name=The Very Hungry Caterpillar, author=Eric Carle)
map: Book(name=Pancakes, Pancakes!, author=Eric Carle)
map: Book(name=The Tiny Seed, author=Eric Carle)
map: Book(name=Do You Want to Be My Friend?, author=Eric Carle)
map: Book(name=The Mixed-Up Chameleon, author=Eric Carle)
map: Book(name=The Very Busy Spider, author=Eric Carle)
map: Book(name=Papa, Please Get the Moon for Me, author=Eric Carle)
map: Book(name=Today Is Monday, author=Eric Carle)
map: Book(name=From Head to Toe, author=Eric Carle)
map: Book(name=Does A Kangaroo Have A Mother, Too?, author=Eric Carle)
map: Book(name=10 Little Rubber Ducks, author=Eric Carle)
map: Book(name=Where Is Baby's Belly Button?, author=Karen Katz)
map: Book(name=No Biting!, author=Karen Katz)
map: Book(name=I Can Share, author=Karen Katz)
```

如果换做sequence实现:
```
fun findBookNamesOfAuthorsUsingSequence(authors: Set<String>): List<String> {
    return books
        .asSequence()
        .filter {
            println("filter: $it")
            it.author in authors
        }
        .map {
            println("map: $it")
            it.name
        }
        .take(3)
        .toList()
}
```
相比上一段代码, 这里只加了`asSequence()`和`toList()`两个操作符.

看console输出:
```
filter: Book(name=Brown Bear, Brown Bear, What Do You See?, author=Eric Carle)
map: Book(name=Brown Bear, Brown Bear, What Do You See?, author=Eric Carle)
filter: Book(name=1, 2, 3 to the Zoo, author=Eric Carle)
map: Book(name=1, 2, 3 to the Zoo, author=Eric Carle)
filter: Book(name=The Very Hungry Caterpillar, author=Eric Carle)
map: Book(name=The Very Hungry Caterpillar, author=Eric Carle)
```
发现找到想要的3本书之后, 对后续元素就不再进行处理了.

而且省略了中间步骤中的集合建立.


## 创建和操作
创建Sequence除了常规能想到的列举元素, 从list或set转换, 还有用方法生成和块生成两种方法.

### 无状态和有状态
序列操作可以分为两种:
* 无状态的.
* 有状态的.

大多数操作都是无状态的, 有状态的操作有:
* 排序操作.
* 区分操作.

### intermediate和terminal操作
如果sequence的操作返回另一个sequence, 那么它是中间的(intermediate), 否则就是终结的(terminal), 比如 `toList()`或`sum()`. 只有在这种终结操作之后, 序列的元素才可以被获取.

如果没有terminal, 任何之前的intermediate操作都不会被执行. 

举例:
```
sequenceOf("Hello", "Kotlin", "World")
    .onEach { println(it) }
```
什么都打印不出来.

而加上`toList()` (terminal操作)之后:
```
sequenceOf("Hello", "Kotlin", "World")
    .onEach { println(it) }
    .toList()
```
就可以把词都打印出来.


`onEach()`是中间操作, 如果想要终结操作, 用`forEach()`, 操作会被执行.
```
sequenceOf("Hello", "Kotlin", "World")
    .forEach { println(it) }
```


这是跟内部实现有关, sequence的intermediate操作符并不做实际的计算, 只是把sequence又包装了一层. (装饰器模式).
在terminal操作的时候, 所有的计算一起进行.

内部实现的具体解释可以看这篇文章: [Inside Sequences: Create Your Own Sequence Operations](https://typealias.com/guides/inside-kotlin-sequences/). 


## 选择和取舍
使用使用sequence的好处是什么呢? 避免了中间结果的建立, 改善性能. 而且, 当我们最后的结果是指定个数的, 取够结果之后就不用再管后面的元素, 节省了操作.

但是在处理比较小的集合或比较简单的操作的时候, lazy的方式也有可能带来问题. 所以究竟用`Sequence`还是`Iterable`还是要根据实际用途和数据来选择的.

影响性能的因素:
* 操作数量. -> 操作数量越多, collection需要创建的中间集合越多. -> `sequence优先`.
* 短路操作. 比如`take()`, `contains()`, `indexOf()`, `any()`, `none()`, `find()`, `first()`. -> `sequence优先`.
* 大小可变的集合. 背景: 数组扩容. `map()`操作时, 因为collection知道数量, 可以直接设置正确的数组大小, 所以不存在消耗. 但是sequence不知道元素数量, 所以`toList()`先创建一个默认大小的数组, 然后随着元素增多, 按需扩容. -> `collection优先`. 
* 有状态的操作, 比如排序, 中间会创建大小可变的集合, 理由同上一条. -> `collection优先`. 

当然终极的方法还是实际测量一下, 才能知道到底有多大区别. 工具: [JMH](https://openjdk.java.net/projects/code-tools/jmh/)

除了性能, 还有其他考虑的方面:
* 生成器. sequence没有size, 可以生成无限长的序列. 在请求的时候生成元素.
* 可用的操作. sequence没有从后到前的操作, 不支持反转, 切片, 集合操作, 获取单个元素的操作等.

## Sequence和Java Stream
Kotlin的sequence和Java 8引入的stream是很像的.

主要的不同点是stream有parallel模式.

## Key Takeaways
sequence的关键点:
* 不创建中间集合.
* 单元素进行所有操作, 之后再到下一个元素.
* 中间操作符是lazy的.

## 参考
* 官方文档: [Sequences](https://kotlinlang.org/docs/reference/sequences.html)

强烈推荐这三篇文章, 里面有个蜡笔工厂的例子很形象, 配图很棒: 
* [Kotlin Sequences: An Illustrated Guide](https://typealias.com/guides/kotlin-sequences-illustrated-guide/): 这篇讲基本.
* [Inside Sequences: Create Your Own Sequence Operations](https://typealias.com/guides/inside-kotlin-sequences/) 这篇讲内部是怎么实现的.
* [When to Use Sequences](https://typealias.com/guides/when-to-use-sequences/) 这篇讲选用时候的考虑因素.

还有这个Effective Kotlin系列中sequence的这篇:
* [Effective Kotlin: Use Sequence for bigger collections with more than one processing step](https://blog.kotlin-academy.com/effective-kotlin-use-sequence-for-bigger-collections-with-more-than-one-processing-step-649a15bb4bf)
