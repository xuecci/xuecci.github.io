---
title: Kotlin Get Started
date: 2020-01-18 01:13:47
tags: Kotlin
categories: 技术
---
Kotlin学习笔记

# 变量
## 变量的声明与赋值
我们回忆下 Java 里声明一个 View 类型的变量的写法：
```
View v;
```
kotlin 里声明一个变量的格式是这样的：
```
var v: View
```

这里有几处不同：
* 有一个 var 关键字
* 类型和变量名位置互换了
* 中间是用冒号分隔的
* 结尾没有分号（对，Kotlin 里面不需要分号）
<!--more-->
## Kotlin 的空安全设计

简单来说就是通过 IDE 的提示来避免调用 null 对象，从而避免 NullPointerException。
```
var view: View = null
// IDE 会提示错误，Null can not be a value of a non-null type View
```
使用？来表示可空，这种类型之后加 ? 的写法，在 Kotlin 里叫可空类型。
```
class User {
    var name: String? = null
}

view?.setBackgroundColor(Color.RED)  //使用？来避免NullPointerException，如果view为空即不会执行setBackgroundColor
view!!.setBackgroundColor(Color.RED) //使用！！来做非空断言，即开发者确保view不为空，编译器不需要检查，如果出现空的情况，运行就会抛出异常
```

## 延迟初始化
在开始无法初始化的对象，Kotlin提供了延迟初始化关键字lateinit
这个 lateinit 的意思是：告诉编译器我没法第一时间就初始化，但我肯定会在使用它之前完成初始化的。
```
lateinit var view: View
override fun onCreate(...) {
    ...
    view = findViewById(R.id.tvContent)
}
```

## 类型推断
```
var name: String = "Mike"
var name = "Mike"
```

## val 和 var
val 和 Java 中的 final 类似，称为只读变量。它只能赋值一次，不能修改。而 var 是一种可读可写变量。
```
val size = 18
```

## 可见性
在 Kotlin 里变量默认是 public 的，其他与Java类似

# 函数
Kotlin 除了变量声明外，函数的声明方式也和 Java 的方法不一样。Java 的方法（method）在 Kotlin 里叫函数（function），其实没啥区别，或者说其中的区别我们可以忽略掉。对任何编程语言来讲，变量就是用来存储数据，而函数就是用来处理数据。

## 函数的声明
```
fun cook(name: String): Food {
    ...
}
```
* 以 fun 关键字开头
* 返回值写在了函数和参数后面，如果没有返回值，写作Unit，并且可以省略

## 可见性
函数如果不加可见性修饰符的话，默认的可见范围和变量一样也是 public 的

## 属性的 getter/setter 函数
我们知道，在 Java 里面的 field 经常会带有 getter/setter 函数
在 Kotlin 里，这种 getter / setter 是怎么运作的呢？
```
class User {
    var name = "Mike"
        get() {
            return field + " nb"
        }
        set(value) {
            field = "Cute " + value
        }
}
```
# 类型
## 基本类型
在 Kotlin 中，所有东西都是对象，Kotlin 中使用的基本类型有：数字、字符、布尔值、数组与字符串。
```
var number: Int = 1 // 👈还有 Double Float Long Short Byte 都类似
var c: Char = 'c'
var b: Boolean = true
var array: IntArray = intArrayOf(1, 2) // 👈类似的还有 FloatArray DoubleArray CharArray 等，intArrayOf 是 Kotlin 的 built-in 函数
var str: String = "string"
```
这里有两个地方和 Java 不太一样：
Kotlin 里的 Int 和 Java 里的 int 以及 Integer 不同，主要是在装箱方面不同。
Java 里的 int 是 unbox 的，而 Integer 是 box 的：
```
int a = 1;
Integer b = 2; // 👈会被自动装箱 autoboxing
Kotlin 里，Int 是否装箱根据场合来定：
```
```
var a: Int = 1 // unbox
var b: Int? = 2 // box
var list: List<Int> = listOf(1, 2) // box
```
Kotlin 在语言层面简化了 Java 中的 int 和 Integer，但是我们对是否装箱的场景还是要有一个概念，因为这个牵涉到程序运行时的性能开销。
因此在日常的使用中，对于 Int 这样的基本类型，尽量用不可空变量。
Java 中的数组和 Kotlin 中的数组的写法也有区别：
```
int[] array = new int[] {1, 2};
```
而在 Kotlin 里，上面的写法是这样的：
```
var array: IntArray = intArrayOf(1, 2)
// 👆这种也是 unbox 的
```
简单来说，原先在 Java 里的基本类型，类比到 Kotlin 里面，条件满足如下之一就不装箱：
* 不可空类型。
* 使用 IntArray、FloatArray 等。

## 类型的判断和强转
Kotlin 里同样有类似解决方案，使用 is 关键字进行「类型判断」，并且因为编译器能够进行类型推断，可以帮助我们省略强转的写法：
```
fun main() {
    var activity: Activity = NewActivity()
    if (activity is NewActivity) {
        // 👇的强转由于类型推断被省略了
        activity.action()
    }
}
```
那么能不能不进行类型判断，直接进行强转调用呢？可以使用 as 关键字：
```
fun main() {
    var activity: Activity = NewActivity()
    (activity as NewActivity).action()
}
```
这种写法如果强转类型操作是正确的当然没问题，但如果强转成一个错误的类型，程序就会抛出一个异常。
我们更希望能进行安全的强转，可以更优雅地处理强转出错的情况。
这一点，Kotlin 在设计上自然也考虑到了，我们可以使用 as? 来解决：
```
fun main() {
    var activity: Activity = NewActivity()
    // 👇'(activity as? NewActivity)' 之后是一个可空类型的对象，所以，需要使用 '?.' 来调用
    (activity as? NewActivity)?.action()
}
```
它的意思就是说如果强转成功就执行之后的调用，如果强转不成功就不执行。