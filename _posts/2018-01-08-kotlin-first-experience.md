--- 
layout: post
title: Kotlin 初体验
category: Kotlin
comments: true
image:
  feature: texture-feature-02.jpg
excerpt: Kotlin 是 JetBrains 公司（著名的 IntelliJ IDEA 正是由这家公司开发的，Android Studio 也是基于 IDEA 的）在 2011 年推出的在 JVM 上运行的静态类型编程语言，2016 年发布了第一个稳定版本， 2017 年 Google I/O 上被 Google 定为 Android 开发一级语言。Kotlin 与 Java 100% 兼容等特性和 Google 官方支持等诸多因素的影响下，开始了对 Kotlin 的探索和使用。
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section>

### 背景
Kotlin 是 JetBrains 公司（著名的 IntelliJ IDEA 正是由这家公司开发的，Android Studio 也是基于 IDEA 的）在 2011 年推出的在 JVM 上运行的静态类型编程语言，2016 年发布了第一个稳定版本， 2017 年 Google I/O 上被 Google 定为 Android 开发一级语言。Kotlin 与 Java 100% 兼容等特性和 Google 官方支持等诸多因素的影响下，开始了对 Kotlin 的探索和使用。

### Kotlin 的魅力所在

#### 更安全

空指针是被认为是价值 billion-dollar 的问题，在日常的开发过程中，不论多么小心，都无法做到完全的避免它。而且为了不发生空指针异常，往往会写很多的防御性代码，各种各样的 if 判断。Kotlin 是空指针安全的，增加了可为空类型和不可为空类型的区别，使用 `?` 来区分。

```
var output: String
output = null   // 对于不可为空类型，如果被赋值 null，就会发生编译错误

val name: String? = null   
println(name.length())      // 对于可为空的类型，直接使用，也会发生编译错误
```

使用 Android Studio 提供的 Show Kotlin Bytecode 工具可以查看字节码来理解其原理，Kotlin 实际上是使用空指针安全符，在内部判空来确保不出现空指针的。例如以下的代码，

```
private fun test(str: String?) {
    print(str?.length)
}
```
翻译成字节码如下，
```
  // access flags 0x12
  private final test(Ljava/lang/String;)V
   L0
    LINENUMBER 10 L0
    ALOAD 1
    DUP
    IFNULL L1 // 对字符串进行判空
    INVOKEVIRTUAL java/lang/String.length ()I
    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
    GOTO L2
   L1
    POP
    ACONST_NULL
   L2
    ASTORE 2
   L3
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    ALOAD 2
    INVOKEVIRTUAL java/io/PrintStream.print (Ljava/lang/Object;)V
   L4
   L5
    LINENUMBER 11 L5
    RETURN
   L6
    LOCALVARIABLE this Lcom/wms/app/Test; L0 L6 0
    LOCALVARIABLE str Ljava/lang/String; L0 L6 1
    MAXSTACK = 2
    MAXLOCALS = 3
```

#### 语法简单，不啰嗦

Java 肯定不是一个简洁的语言，甚至有一些啰嗦，并不能说这是一个缺点，但太多的冗余可能会导致一些潜在的 Bug。Kotlin 在这方面做了一些工作，使得代码可以变得更加简洁。

Kotlin 的简洁体现在以下几个方面：

* 支持类型推断，不需要显式的指明变量的类型
* var 表示变量，val 表示常量
* 省略每行语句后的分号
* 新增 data class，创建一个数据类，不需要手动添加 getter，setter，equals()，hashCode()，toString() 和 copy() 方法
* 支持 lambda 表达式，不用写匿名的内部类了
* 引入参数的默认值，可以少写很多重载方法
* 支持扩展，各种各样的 utils 工具类都可以通过扩展来实现了
* 类型智能转换
* 字符串模板
* 区间表达式
* 等等等

```
class TestActivity : BaseActivity() {

    private var mNum = 2 // 不需要显示指明 mNum 是 Int 类型变量
    private val mName = "test" // val 常量

    private fun initViews() {
        button.setOnClickListener { // lambda 表达式，不需 new View.OnClickListener()
            showToast("hello $mName") // 字符串模板
        }
    }

}

data class User(var username: String, var age: Int, var avatar: String) // data class，一行代码可以搞定 Java 中的一个类

fun Context.showToast(msg: String?, duration: Int = Toast.LENGTH_SHORT) { // 扩展方法，并且提供默认参数 duration，调用方可以省略 duration 使用默认值
    Toast.makeText(this, msg, duration).show()
}
```

#### 再也不用 findViewById

在 Android 开发过程中，要使用一个 View 的时候，常规的方式是需要通过 findViewById 来找到这个 View。因为不愿意重复繁琐的工作，有很多的插件及开源库 ButterKnife 来帮助开发者减免这部分工作。在 Kotlin 中，可以使用 Kotlin 的 Android 插件来直接使用视图 id 来操作视图，不再需要任何的 findViewById 了。摘自官网的例子如下：

```
import kotlinx.android.synthetic.main.content_main.*

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // No need to call findViewById(R.id.textView) as TextView
        textView.text = "Kotlin for Android rocks!"
    }
}
```

#### 与 Java 的交互性

Kotlin 是基于 JVM 的语言，编译成 bytecode，与 Java 交互性好。Kotlin 和 Java 可以互调，只需要注意一下其中的差异即可。甚至可以一键将 Java 代码转化成 Kotlin。

#### 性能

Kotlin 的性能理论上是和 Java 相当的，甚至可能会略好一些。具体的性能数据可以参考 [Benchmarks 文章](https://sites.google.com/a/athaydes.com/renato-athaydes/posts/kotlinshiddencosts-benchmarks)。总体来说，Kotlin 存在比 Java 性能更好的地方，例如 `lambda` 快一些，`companion object` 比 Java 的静态常量的访问快一些；也有性能更差的部分，对于 `varargs` 参数的展开需要较高的性能开销。

### 使用情况

目前，Kotlin 已经在 Slack，Evernote，Pinterest 等等应用中应用到了生产中。从各方面的表现来看，在生产环境使用 Kotlin 已经不存在什么显著问题了。我在项目中从四个月前开始引入 Kotlin，两个月前逐渐开始大规模使用。从 App 层的代码统计结果来看，Kotlin 占比已经达到了 69%（12129 / (12129 + 5446)）。在整个项目中，Kotlin 的占比 20% 左右。这些百分比只计算 Java 和 Kotlin 代码，UI 层的 XML 代码和其他脚本代码均不包含在内。

从开发效率的角度，Kotlin 比 Java 高效一些，总体上可以节约 20% 左右代码量。实现相同功能的 Java 代码和 Kotlin 代码也可以进行一个比较，同样是实现一个功能，Java 代码 665 行，Kotlin 代码 497 行，Kotlin 减少了 25%。

#### 存在的问题

* 空指针安全符在实际的使用中并不是万能的

在某些业务场景下，必要的 if 判空仍并不可少，在业务上为空的情况时常需要处理。同理，从外界接收的数据也不能使用不可为空类型，因为不能保证服务端的数据是否会传空值。

* 扩展方法可能被滥用

扩展方法很好用，可以对已有的类增加新的方法，简便实现系统层不支持的功能，在使用中需要谨慎小心，不能违背类的职责，需要考虑添加的合理范围，不能仅仅为了方便使用而增加一些不因该类实现的方法。

#### 最后的小结

在很多人看来，Kotlin 可能只是语法糖的堆砌。语法糖存在的目的就是为了让代码变得简洁易读，这本身是件好事情，确实可以切实的减少代码量。而且 Kotlin 使用语法糖的另外一个原因是 Kotlin 是支持到 Java 6 的 JVM 的， invokeddynamic 不是所有版本的 JVM 都支持的，Kotlin 的 lambda 表达式的实现实际上还是使用匿名内部类来实现的。

不管任何的开发语言都只是工具，如何利用工具来提升工作效率是每个开发者都需要考虑的事情。好的工具，可以显著的提高开发效率。借用 Google I/O 中 Kotlin 的布道中的一句话来结束，「Life is Great and Everything will be Ok, Kotlin is here」，希望更多的开发者和更多的团队可以接受她并且爱上她，共同为提升开发效率而努力。

### 参考
* [Kotlin and Android](https://developer.android.com/kotlin/index.html)
* [Kotlin 官网](https://kotlinlang.org)
* [Kotlin's hidden costs - Benchmarks](https://sites.google.com/a/athaydes.com/renato-athaydes/posts/kotlinshiddencosts-benchmarks)



