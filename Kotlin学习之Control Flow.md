# Kotlin学习之Control Flow



## if

关于if的使用Kotlin与java大同小异，除了传统的使用方式，Kotlin还支持一些额外的特性。

在Kotlin中没有三元运算符 ? :，因为if可以作为表达式来实现这个功能。

```kotlin
val max = if (a > b) a else b

var max: Int
if (a > b) {
    max = a
} else {
    max = b
}

val max = if (a > b) {
    print("Choose a")
    a
} else {
    print("Choose b")
    b
}
```

以上三段代码结果是一样的

需要注意的是，如果if作为一个表达式最终为返回一个结果或为变量赋值，这个表达式一定要有else。否则编译器会报错。



## When

Kotlin中用when来代替switch。和if一样，when也可以用作表达式，当作为表达式使用时，除非编译器能识别出所有的情况都被覆盖到，不然else是分支必须存在。

```kotlin
fun whenAsExpression() {
    var a = 1
    var x = when (a) {
        in 0..1 -> 2
        2,4,5 -> 3
        else -> 10
    }
    println("x=$x")
}

fun whenAsStatement() {
    var a = 100
    when (a) {
        in 1..5 -> println("in 1..5")
        !in 1..5 -> println("not in 1..5")
    }
}
```

when当中也支持布尔值，当值为`true`时，就会进入该分支。

e.g.可以用is作类型判断(类似于java中的instance of)

```kotlin
fun hasPrefix(x: Any) = when(x) {
    is String -> x.startsWith("prefix")
    else -> false
}
```



## For循环

for能遍历任何可以提供迭代器的容器。



```kotlin
fun testFor() {
    var array = arrayOf("a", "b", "c")

    for(i in array.indices){
        print(array[i] + " ")
    }

    println()

    for ((index, value) in array.withIndex()) {
        println("index=$index | value=$value")
    }
}
```

*如果用for循环遍历数组，编译其会把代码编译成基于索引值的for循环而不是创建一个迭代器。*



## While循环

支持while和do...while

```kotlin
while (x > 0) {
    x--
}

do {
    x++
} while (x < 10)
```



## Return and Jumps

在Kotlin中，控制流基本大同小异。但在跳转表达式配上标记上却有些与众不同的特性。

Kotlin中的所有表达式都能标记，将标记符放在表达式之前即可，标记的语法格式为：`labelName@`



```kotlin
fun breakWithLabel() {

    loop@ for (i in 1..5) {
        println("i=$i")
        for (j in 1..5) {
            if (j == 2)
                break@loop
            else
                println("j=$j")
        }
    }


}

fun breakWithoutLabel() {

    for (i in 1..5) {
        println("i=$i")
        for (j in 1..5) {
            if (j == 2)
                break
            else
                println("j=$j")
        }
    }
}
```

第一个方法在执行到`break`时会直接跳出外层的for循环，而第二个方法只会跳出内层的for循环。