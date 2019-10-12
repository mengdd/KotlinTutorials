# Kotlin集合

## 集合创建
Kotlin标准库提供了set, list和map这三种基本集合类型的实现, 每种类型又都分为可变和不可变(只读)两种类型.

创建不同类型的集合:
```
// set
val numbersSet = setOf("one", "two", "three", "four")
val numbersSetMutable = mutableSetOf("one", "two", "three", "four")

// list
val numbersList = listOf("one", "two", "three", "four")
val numbersListMutable = mutableListOf("one", "two", "three", "four")

// map
val numbersMap = mapOf("key1" to 1, "key2" to 2, "key3" to 3, "key4" to 1)
val numbersMapMutable = mutableMapOf("key1" to 1, "key2" to 2, "key3" to 3, "key4" to 1)
```
其中不可变的类型不支持添加, 删除和更改元素.


标准库为每个集合都提供了empty的只读单例:
```
val emptySet = emptySet<String>()
val emptyList = emptyList<String>()
val emptyMap = emptyMap<String, Int>()
```
可以用于在某种情况下返回空结果.

### 集合引用赋值和拷贝
可以通过赋值把mutable的变量付给只读的引用, 从而禁止其修改行为. 比如传递函数参数等.
```
val sourceList = mutableListOf(1, 2, 3)
val referenceList: List<Int> = sourceList
//referenceList.add(4)            //compilation error
sourceList.add(4)
println(sourceList)
println(referenceList) // shows the current state of sourceList
```
但是对源数据的修改仍然会反映到新的只读引用上, 它们的数据保持同步.

`toList`, `toSet`会拷贝元素, 创建新的集合. 之后对源的修改不会作用于拷贝的集合上, 反之亦然.
```
val sourceList = mutableListOf(1, 2, 3)
val copyList = sourceList.toList()
sourceList.add(4)
println(sourceList)
println(copyList) // is different from current sourceList
```

还有`toMutableList()`和`toMutableSet()`, 除了可以进行可变/不可变的转换以外, 还可以进行set和list之间的相互转换.

## 遍历集合
遍历集合是比较常见的操作. 举个例子, 现在有一个list, 里面的元素类型是`Book`, 我想遍历这个集合, 输出所有的书名.
```
data class Book(val name: String, val author: String)
```
做法很多, 这里列出比较常规的几种写法:
```
// way 1
for (book in books) {
    println(book.name)
}

// way 2
books.forEach { println(it.name) }

// way 3
val iterator = books.iterator()
while (iterator.hasNext()) {
    println(iterator.next().name)
}

// way 4
for (i in books.indices) {
    println(books[i].name)
}
```
这里数据结构是list, 所以可以用`[]`加索引的方式访问.

## 最有用/常用的操作符
Kotlin提供了很多有用的集合操作符, 之前用Java的时候借助RxJava来做的一些集合元素过滤转换排序等等, 现在Kotlin的标准库就支持了. (用过RxJava的同学可以感觉到这里学习起来很平滑.)

这里举例说明一下比较常用的场景.

### `filter`和`map`
过滤操作用于筛选符合条件的元素, 比如在很多书中查找某一个作者的书:
```
fun getBooksOfAuthor(books: List<Book>, author: String): List<Book> {
    return books.filter { it.author == author }
}
```
这里返回的仍然是书的列表, 如果我想返回书名的列表呢?
```
fun getBooksNamesOfAuthor(books: List<Book>, author: String): List<String> {
    return books
        .filter { it.author == author }
        .map { it.name }
}
```
这里利用map做了一个转换, `Book`变成了`String`, 最后返回的是`List<String>`.

### `map`和`flatMap`
`map`和`flatMap`都是对集合中每个元素进行转换, 它们有什么区别呢?

这里还是通过例子来看, 除了前面的`Book`之外, 我们的数据类型增加了`Student`和`Order`:
```
data class Student(val name: String, val orders: List<Order>)

data class Order(val books: List<Book>)
```
场景: 学生要买书, 在网站上下订单, 每个人可以下多个订单, 订单里又可以有多本书.

求解:
* 问题1: 某一个学生都定了什么书?
* 问题2: 有多个学生, 要所有的学生定的书的集合?

针对第一个问题, 是这样解决的:
```
fun getBooksOrderedByStudent(student: Student): Set<Book> {
    return student.orders.flatMap { it.books }.toSet()
}
```
这里用的`flatMap`, 如果改成`map`呢?
```
fun getBooksListsOrderedByStudent(student: Student): Set<List<Book>> {
    return student.orders.map { it.books }.toSet()
}
```
可以看到返回值变了, 不再是书的集合`Set<Book>`了, 而是`Set<List<Book>>`. 这不是我们想要的答案.


二者的区别点进源码就可以发现:
```
public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.mapTo(destination: C, transform: (T) -> R): C {
    for (item in this)
        destination.add(transform(item))
    return destination
}

public inline fun <T, R, C : MutableCollection<in R>> Iterable<T>.flatMapTo(destination: C, transform: (T) -> Iterable<R>): C {
    for (element in this) {
        val list = transform(element)
        destination.addAll(list)
    }
    return destination
}
```
虽然都是进行转换, `map`的转换是一对一的, 元素的个数不会变; 而`flatMap`的转换是一对多的, 转换后元素变成了集合, 有一个铺平的过程, 转换后生成的集合元素会被合并返回.

想明白了`flatMap`的用法, 解决问题2就比较容易了. 有多个学生, 那么我们需要统计每个学生订单里的书:
```
fun getAllBooksOrderedByAllStudents(students: List<Student>): Set<Book> {
    return students.flatMap { student ->
        student.orders.flatMap { order ->
            order.books
        }
    }.toSet()
}
```

### 排序和查找
实际的应用里可能还有搜索排序等需求.

排序, 可以`sortedBy()`指定根据哪个字段, 也可以用`sortedWith()`写一个自定义的`Comparator`.
这里代码例子, 比如按书名/书名长度排序:
```
fun sortBooksByName(books: List<Book>): List<Book> {
    return books.sortedBy { it.name }
}

fun sortBooksByNameLength(books: List<Book>): List<Book> {
    return books.sortedBy { it.name.length }
}

fun sortBooksByComparator(books: List<Book>): List<Book> {
    return books.sortedWith(
        Comparator { book1, book2 ->
            return@Comparator book1.name.length - book2.name.length
        }
    )
}
```
还有可能会有查找的需求, 查找是否存在元素, 返回第一个元素等等.

是否存在某个作者的书:
```
fun hasBookOfAuthor(books: List<Book>, author: String): Boolean {
    return books.any { it.author == author }
}
```
返回的是布尔值.

查找某个作者的书, 返回第一个结果:
```
fun findOneBookOfAuthor(books: List<Book>, author: String): Book? {
    return books.find { it.author == author }
}
```
注意这里返回的类型是`Book?`, 如果没有符合条件的会返回null.


查找某个实例是否在集合中, 除了用`contains()`之外, 还可以用`in`:
```
if (bookBrownBear in books) {
    println("we have book Brow Bear")
}
```

其他有用的操作符比如`first()`和`firstOrNull`, 还有统计个数的`count()`.

## 其他有用/有意思的东东
学习一个语言的最好方法是边学边用, 在看别人的代码的时候总是能发现一些新东西.

 
下面列出几点:
* 可以在遍历的时候删除元素了. -> `MutableIterator. `
* 根据某一标准进行分组, 把list转换成map. -> `groupBy()`. 注意和`associateBy()`的区别: `associateBy`会把符合条件的最后一个值作为value, 而`groupBy()`的value是list.
* 更加方便的获取元素的方法: `elementAtOrElse()`, `elementAtOrNull()`, `getOrElse()`, `getOrDefault()`.

集合这部分的知识点实在太多, 也很细节, 无法一一列举, 感兴趣的话可以看看官方文档.

## 参考
* 官网: [Kotlin Collections Overview](https://kotlinlang.org/docs/reference/collections-overview.html)
* Kotlin练习: [Kotlin Koans](https://play.kotlinlang.org/koans/overview)
* Learn Kotlin by Example: [Learn Kotlin by Example](https://play.kotlinlang.org/byExample/05_Collections/01_List)