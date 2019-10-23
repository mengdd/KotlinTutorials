# Kotlin Extensions
Kotlin提供了extensions, 用于扩展类的功能, 而不用继承或者装饰这个类.

有扩展方法(extension functions), 也有扩展属性(extension properties), 使用的时候就跟这个类的一般方法和属性一样.

查看反编译的java代码可以发现extensions对应的是静态方法, 所以本质上是一个语法糖, 只是为了用起来更直观和方便.


## 扩展方法 Extension functions
### 定义
定义扩展方法的时候, 在方法前面加上它的接受者(receiver)类型即可.

比如先有一个`Book`类型:
```
data class Book(val name: String, val author: String)
```
可以定义扩展方法:
```

fun Book.getFullName(): String {
    return this.name + " by " + this.author
}
```

扩展方法中的`this`关键字就对应接受者对象(在`.`之前传入的那个对象).

扩展方法使用的时候和成员方法一样:
```
val book = Book("My Car", "Byron Barton")
println(book.getFullName())
```

定义一个extension并不会修改被扩展的类, 只是在这个类型的变量上增加了用`.`来调用这个方法的方式.

扩展属性的定义类似:
```
val Book.nameLength: Int
    get() = this.name.length
```

扩展方法可以用来给不能修改的代码, 或者是第三方库增加功能.

比如给原生类型增加扩展函数, 判断奇偶性:
```
fun checkEvenAndOdd(): List<Boolean> {
    val isEven: Int.() -> Boolean = { this % 2 == 0 }
    val isOdd: Int.() -> Boolean = { this % 2 != 0 }
    return listOf(42.isOdd(), 239.isOdd(), 294823098.isEven())
}
```

### 静态解析
* 扩展方法是静态解析的. (会以声明类型而不是运行时类型为准.)
* 如果类有同名成员变量, 那么**成员变量永远优先**.
但是同名扩展方法如果提供不一样的方法签名, 可以实现对成员变量方法的重载.

### 可空的receiver和泛型
receiver的类型可以是nullable的.
比如这个可空的扩展方法:
```
fun Book?.nullSafeToString() = this?.toString() ?: "NULL"
```

类型还可以是泛型, 所以还可以把上面这个方法改成泛型的:
```
fun <T> T?.nullSafeToString() = this?.toString() ?: "NULL"
```

## Android开发中的应用
### Android KTX
Android KTX是官方提供的一套扩展库, 提供了更简化的方式来写代码.

比较常用的比如:

fragment的transaction:
```
fragmentManager().commit {
   addToBackStack("...")
   setCustomAnimations(
           R.anim.enter_anim,
           R.anim.exit_anim)
   add(fragment, "...")
}
```

做数据库的transaction:
```
db.transaction {
    // insert data
}
```

shared preferences的操作:
```
haredPreferences.edit(commit = true) { putBoolean("key", value) }
```

### BindingAdapters
在使用data binding的时候经常需要用`@BindingAdapter`自定义属性.

通常的做法是这样, 把view作为第一个参数:
```
@BindingAdapter("imageUrl")
fun setImageUrl(imageView: ImageView, url: String?) {
    Glide.with(imageView.context).load(url).into(imageView)
}
```
但是也可以写成扩展函数的形式:
```
@BindingAdapter("imageUrl")
fun ImageView.setImageUrl(url: String?) {
    Glide.with(context).load(url).into(this)
}
```
这样在代码中可以直接在实例上调用:
```
myImageView.setImageUrl("myUrl")
```

## 参考
* 官方文档: [Extensions](https://kotlinlang.org/docs/reference/extensions.html)
* Android KTX: [Android KTX](https://developer.android.com/kotlin/ktx)
* [[Kotlin Pearls 6] Extensions: The Good, The Bad and The Ugly](https://proandroiddev.com/kotlin-pearls-6-extensions-the-good-the-bad-and-the-ugly-23c88fcab23)