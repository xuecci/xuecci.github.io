---
title: Kotlin Chapter Two
date: 2020-01-19 15:47:52
tags: Kotlin
---
# 类和对象
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
    }
}
```
* 首先是类的可见性，Java 中的 public 在 Kotlin 中可以省略，Kotlin 的类默认是 public 的。
* 类的继承的写法，Java 里用的是 extends，而在 Kotlin 里使用 :，但其实 : 不仅可以表示继承，还可以表示 Java 中的 implement

Kotlin 里我们注意到 AppCompatActivity 后面的 ()，这其实也是一种省略的写法，等价于：
```
class MainActivity constructor() : AppCompatActivity() {
}
或者
// 注意这里 AppCompatActivity 后面没有 '()'
class MainActivity : AppCompatActivity {
    constructor() {
    }
}
```
Kotlin 把构造函数单独用了一个 constructor 关键字来和其他的 fun 做区分。

override 的不同
* Java 里面 @Override 是注解的形式。
* Kotlin 里的 override 变成了关键字。
* Kotlin 省略了 protected 关键字，也就是说，Kotlin 里的 override 函数的可见性是继承自父类的。
除了以上这些明显的不同之外，还有一些不同点从上面的代码里看不出来，但当你写一个类去继承 MainActivity 时就会发现：
Kotlin 里的 MainActivity 无法继承：
```
// 👇写法会报错，This type is final, so it cannot be inherited from
class NewActivity: MainActivity() {
}
```
原因是 Kotlin 里的类默认是 final 的，而 Java 里只有加了 final 关键字的类才是 final 的。
那么有什么办法解除 final 限制么？我们可以使用 open 来做这件事：
```
open class MainActivity : AppCompatActivity() {}
```
这样一来，我们就可以继承了。
```
class NewActivity: MainActivity() {}
但是要注意，此时 NewActivity 仍然是 final 的，也就是说，open 没有父类到子类的遗传性。
而刚才说到的 override 是有遗传性的：
```
class NewActivity : MainActivity() {
    // 👇onCreate 仍然是 override 的
    override fun onCreate(savedInstanceState: Bundle?) {
        ...
    }
}
```
如果要关闭 override 的遗传性，只需要这样即可：
```
open class MainActivity : AppCompatActivity() {
    // 👇加了 final 关键字，作用和 Java 里面一样，关闭了 override 的遗传性
    final override fun onCreate(savedInstanceState: Bundle?) {
        ...
    }
}
```

# 类型的判断和强转
```
fun main() {
    var activity: Activity = NewActivity()
    if (activity is NewActivity) {
        // 👇的强转由于类型推断被省略了
        activity.action()
    }
}
//使用as关键字来强转
fun main() {
    var activity: Activity = NewActivity()
    (activity as NewActivity).action()
}
//使用as?来避免空对象，避免报错
fun main() {
    var activity: Activity = NewActivity()
    // 👇'(activity as? NewActivity)' 之后是一个可空类型的对象，所以，需要使用 '?.' 来调用
    (activity as? NewActivity)?.action()
}
```
