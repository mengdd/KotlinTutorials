# Kotlin DSL for HTML实例解析
Kotlin DSL, 指用Kotlin写的Domain Specific Language.
本文通过解析官方的Kotlin DSL写html的例子, 来说明Kotlin DSL是什么.

首先是一些基础知识, 包括什么是DSL, 实现DSL利用了那些Kotlin的语法, 常用的情形和流行的库.

对html实例的解析, 没有一冲上来就展示正确答案, 而是按照分析需求, 设计, 和实现细化的步骤来逐步让解决方案变得明朗清晰.

## 理论基础
### DSL: 领域特定语言
DSL: Domain Specific Language.
专注于一个方面而特殊设计的语言.

可以看做是封装了一套东西, 用于特定的功能, 优势是复用性和可读性的增强. -> 意思是提取了一套库吗? 

不是.

DSL和简单的方法提取不同, 有可能代码的形式或者语法变了, 更接近自然语言, 更容易让人看懂.

### Kotlin语言基础
做一个DSL, 改变语法, 在Kotlin中主要依靠:
* lambda表达式.
* 扩展方法.

三个lambda语法:
* 如果只有一个参数, 可以用`it`直接表示.
* 如果lambda表达式是函数的最后一个参数, 可以移到小括号`()`外面. 如果lambda是唯一的参数, 可以省略小括号`()`.
* lambda可以带receiver.

扩展方法.

### 流行的DSL使用场景
Gradle的build文件就是用DSL写的.
之前是Groovy DSL, 现在也有Kotlin DSL了.


还有[Anko](https://github.com/Kotlin/anko).
这个库包含了很多功能, UI组件, 网络, 后台任务, 数据库等.

和服务器端用的: [Ktor](https://ktor.io/servers/features/routing.html)


应用场景: Type-Safe Builders
type-safe builders指类型安全, 静态类型的builders.

这种builders就比较适合创建Kotlin DSL, 用于构建复杂的层级结构数据, 用半陈述式的方式.

官方文档举的是html的例子.
后面就对这个例子进行一个梳理和解析.

## html实例解析
### 1 需求分析
首先明确一下我们的目标.

做一个最简单的假设, 我们期待的结果是在Kotlin代码中类似这样写:
```kotlin
html {
    head { }
    body { }
}
```
就能输出这样的文本:
```html
<html>
  <head>
  </head>
  <body>
  </body>
</html>
```

#### 发现1: 调用形式
仔细观察第一段Kotlin代码, `html{}`应该是一个方法调用, 只不过这个方法只有一个lambda表达式作为参数, 所以省略了`()`.

里面的`head{}`和`body{}`也是同理, 都是两个以lambda作为唯一参数的方法.

#### 发现2: 层级关系
因为标签的层级关系, 可以理解为每个标签都负责自己包含的内容, 父标签只负责按顺序显示子标签的内容.

#### 发现3: 调用限制
由于`<head>`和`<body>`等标签只在`<html>`标签中才有意义, 所以应该限制外部只能调用`html{}`方法, `head{}`和`body{}`方法只有在`html{}`的方法体中才能调用.

#### 发现4: 应该需要完成的
* 如何加入和显示文字.
* 标签可能有自己的属性.
* 标签应该有正确的缩进.

### 2 设计
#### 标签基类
因为标签看起来都是类似的, 为了代码复用, 首先设计一个抽象的标签类`Tag`, 包含:
* 标签名称.
* 一个子标签的list.
* 一个属性列表.
* 一个渲染方法, 负责输出本标签内容(包含标签名, 子标签和所有属性).

#### 怎么加文字
文字比较特殊, 它不带标签符号`<>`, 就输出自己.
所以它的渲染方法就是输出文字本身.

可以提取出一个更加基类的接口`Element`, 只包含渲染方法. 这个接口的子类是`Tag`和`TextElement`.

有文字的标签, 如`<title>`, 它的输出结果:
```html
    <title>
      HTML encoding with Kotlin
    </title>
```
文字元素是作为标签的一个子标签的.
这里的实现不容易自己想到, 直接看后面的实现部分揭晓答案吧.

### 3 实现
有了前面的心路历程, 再来看实现就能容易一些.

#### 基类实现
首先是最基本的接口, 只包含了渲染方法:
```
interface Element {
    fun render(builder: StringBuilder, indent: String)
}
```

它的直接子类标签类:
```
abstract class Tag(val name: String) : Element {
    val children = arrayListOf<Element>()
    val attributes = hashMapOf<String, String>()

    protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
        tag.init()
        children.add(tag)
        return tag
    }

    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent<$name${renderAttributes()}>\n")
        for (c in children) {
            c.render(builder, indent + "  ")
        }
        builder.append("$indent</$name>\n")
    }

    private fun renderAttributes(): String {
        val builder = StringBuilder()
        for ((attr, value) in attributes) {
            builder.append(" $attr=\"$value\"")
        }
        return builder.toString()
    }

    override fun toString(): String {
        val builder = StringBuilder()
        render(builder, "")
        return builder.toString()
    }
}
```
完成了自身标签名和属性的渲染, 接着遍历子标签渲染其内容. 注意这里为所有子标签加上了一层缩进.

`initTag()`这个方法是`protected`的, 供子类调用, 为自己加上子标签.

#### 带文字的标签
带文字的标签有个抽象的基类:
```
abstract class TagWithText(name: String) : Tag(name) {
    operator fun String.unaryPlus() {
        children.add(TextElement(this))
    }
}
```
这是一个对`+`运算符的重载, 这个扩展方法把字符串包装成`TextElement`类对象, 然后加到当前标签的子标签中去.

`TextElement`做的事情就是渲染自己:
```
class TextElement(val text: String) : Element {
    override fun render(builder: StringBuilder, indent: String) {
        builder.append("$indent$text\n")
    }
}
```
所以, 当我们调用:
```
html {
    head {
        title { +"HTML encoding with Kotlin" }
    }
}
```
得到结果:
```
<html>
  <head>
    <title>
      HTML encoding with Kotlin
    </title>
</html>
```
其中用到的`Title`类定义:
```
class Title : TagWithText("title")
```
通过'+'运算符的操作, 字符串: "HTML encoding with Kotlin"被包装成了`TextElement`, 他是title标签的child.

#### 程序入口
对外的公开方法只有这一个:
```
fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```
`init`参数是一个函数, 它的类型是`HTML.() -> Unit`. 这是一个带接收器的函数类型, 也就是说, 需要一个`HTML`类型的实例来调用这个函数.

这个方法实例化了一个`HTML`类对象, 在实例上调用传入的lambda参数, 然后返回该对象.

调用此lambda的实例会被作为`this`传入函数体内(`this`可以省略), 我们在函数体内就可以调用`HTML`类的成员方法了.

这样保证了外部的访问入口, 只有:
```
html {
    
}
```
通过成员函数创建内部标签.

#### HTML类
HTML类如下:
```
class HTML : TagWithText("html") {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)

    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}
```
可以看出`html`内部可以通过调用`head`和`body`方法创建子标签, 也可以用`+`来添加字符串.

这两个方法本来可以是这样:
```
fun head(init: Head.() -> Unit) : Head {
    val head = Head()
    head.init()
    children.add(head)
    return head
}

fun body(init: Body.() -> Unit) : Body {
    val body = Body()
    body.init()
    children.add(body)
    return body
}
```
由于形式类似, 所以做了泛型抽象, 被提取到了基类`Tag`中, 作为更加通用的方法:
```
protected fun <T : Element> initTag(tag: T, init: T.() -> Unit): T {
    tag.init()
    children.add(tag)
    return tag
}
```
做的事情: 创建对象, 在其之上调用init lambda, 添加到子标签列表, 然后返回.

其他标签类的实现与之类似, 不作过多解释.

### 4 修Bug: 隐式receiver穿透问题
以上都写完了之后, 感觉大功告成, 但其实还有一个隐患.

我们居然可以这样写:
```
html {
    head {
        title { +"HTML encoding with Kotlin" }
        head { +"haha" }
    }
}
```
在head方法的lambda块中, html块的receiver仍然是可见的, 所以还可以调用`head`方法.
显式地调用是这样的:
```
this@html.head { +"haha" }
```
但是这里`this@html.`是可以省略的.

这段代码输出的是:
```
<html>
  <head>
    haha
  </head>
  <head>
    <title>
      HTML encoding with Kotlin
    </title>
  </head>
</html>
```
最内层的haha反倒是最先被加到html对象的孩子列表里.

这种穿透性太混乱了, 容易导致错误, 我们能不能限制每个大括号里只有当前的对象成员是可访问的呢? -> 可以.

为了解决这种问题, Kotlin 1.1推出了管理receiver scope的机制, 解决方法是使用`@DslMarker`.

html的例子, 定义注解类:
```
@DslMarker
annotation class HtmlTagMarker
```
这种被`@DslMarker`修饰的注解类叫做`DSL marker`.

然后我们只需要在基类上标注:
```
@HtmlTagMarker
abstract class Tag(val name: String)
```
所有的子类都会被认为也标记了这个marker.

加上注解之后隐式访问会编译报错:
```
html {
    head {
        head { } // error: a member of outer receiver
    }
    // ...
}
```
但是显式还是可以的:
```
html {
    head {
        this@html.head { } // possible
    }
    // ...
}
```
只有最近的receiver对象可以隐式访问.

## 总结
本文通过实例, 来逐步解析如何用Kotlin代码, 用半陈述式的方式写html结构, 从而看起来更加直观. 这种就叫做DSL.

Kotlin DSL通过精心的定义, 主要的目的是为了让使用者更加方便, 代码更加清晰直观.

## 参考
* 官方文档: [Type-Safe Builders](https://kotlinlang.org/docs/reference/type-safe-builders.html)
* [Domain-Specific Languages In Kotlin](https://www.raywenderlich.com/2780058-domain-specific-languages-in-kotlin-getting-started)

More resources:
* [Kotlin之美——DSL篇](https://www.jianshu.com/p/f5f0d38e3e44)
* [From Java Builders to Kotlin DSLs](https://kotlinexpertise.com/java-builders-kotlin-dsls/)
* [Oversimplified network call using Retrofit, LiveData, Kotlin Coroutines and DSL](https://proandroiddev.com/oversimplified-network-call-using-retrofit-livedata-kotlin-coroutines-and-dsl-512d08eadc16)