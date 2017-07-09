# Kotlin学习之Basis Types



## 前言

本文主要是对官方文档的翻译整理，同时加上一些自己的理解，如有不足，欢迎指出。英语好的推荐直接阅读原版官方文档。

[官方文档地址]: http://kotlinlang.org/docs/reference/



## 数字类型

| Type   | BitWidth |
| ------ | -------- |
| Double | 64       |
| Float  | 32       |
| Long   | 64       |
| Int    | 32       |
| Short  | 16       |
| Byte   | 8        |



### 常量的书写

123 - Int（默认）

123L - Long

0x0F - 16进制

0b00001011 - 2进制

123.5 - Double(默认)

123.5F - Float

123.5f - Float

PS:不支持书写八进制常量



### 下划线在数值中的书写

Kotlin(1.1版本之后)可以在撸代码的时候用下划线让数字的可读性看起来更好。(java7中也支持这一特性)

```kotlin
val oneMillion = 1_000_000
val creditCardNumber = 1234_5678_9012_3456L
val socialSecurityNumber = 999_99_9999L
val hexBytes = 0xFF_EC_DE_5E
val bytes = 0b11010010_01101001_10010100_10010010
```

PS：下划线不能放在数字最前最后或者对类型的声明前后



### 自动装箱

当声明变量可能为null时，需要按以下格式声明

Var/val 变量名 : 类型？ =  初始值

赋予初始值时，会自动装箱，*生成一个对象*。自动装箱不会保证相同对象，但会保证数值相同。

```kotlin
 val a = 10000
 val boxedA: Int? = a
 val anotherA: Int = a
 println(boxedA === anotherA)
 println(boxedA === a)
 println(a === anotherA)
```

```kotlin
false
false
true
```



### 类型转换

在Java中，除了String外的基本类型是支持互相赋值时类型转换的，当然其中分为上转型和下转型。比如将一个int的变量赋值给一个double类型的变量。位数小的可以自动转换为位数大的数据类型

`byte,short,char → int → long → float → double`

然而在Kotlin中并不支持。但kotlin为每个数字类型都提供了转换方法。

- `toByte(): Byte`

- `toShort(): Short`

- `toInt(): Int`

- `toLong(): Long`

- `toFloat(): Float`

- `toDouble(): Double`

- `toChar(): Char`

  ​

  当然，在Kotlin中还是存在“自动类型”转换的。编译后会重载+运算符来保证语义正确。

  ```kotlin
  val l = 1L + 3 // Long + Int => Long
  ```



### 位运算(只支持Int和Long)

- `shl(bits)` – 左位移 (Java's `<<`)
- `shr(bits)` – 右位移 (Java's `>>`)
- `ushr(bits)` – 无符号右位移 (Java's `>>>`)
- `and(bits)` – 与运算
- `or(bits)` – 或运算
- `xor(bits)` – 异或运算
- `inv()` – 非运算





## 字符型与布尔型

### 字符型

上文中已经提到，kotlin中是不支持隐式的类型转换。所以要把字符转为整型，需要显示的调用`toInt(): Int`方法。字符型支持以下的特殊字符 `\t`, `\b`, `\n`, `\r`, `\'`, `\"`, `\\` and `\$`，对于其他字符，可以采用Unicode编码。

### 布尔型

对于可能为null的情况同样会自动装箱。跟Java中的Boolean一样。

- `||` – lazy disjunction
- `&&` – lazy conjunction
- `!` - negation



## 数组

创建一个数组可以调用`arrayOf()`方法或者调用`arrayOfNulls()`方法，后者创建一个给定长度的数组，其中所有引用值都为null。

也可以调用工厂方法

```kotlin
// Creates an Array<String> with values ["0", "1", "4", "9", "16"]
val asc = Array(5, { i -> (i * i).toString() })
```

在Java中，万物皆Object。但在Kotlin中，为了避免运行时错误，是不能将Array<String>赋值给Array<Any>的。

在Kotlin中，还定义了一些基本数据类型的数组。比如`ByteArray`,`ShortArray`,`IntArray`。这些类与`Array`之间没有继承关系。

## 字符串

### 转义字符串(escaped string)与原生字符串(raw string)

Kotlin中的String与Java语言中的一样，都是不可变的并且由一组字符组成。可以通过数组下标的形式访问。

但在Kotlin中有两种类型的字符串，转义字符串可以包含转义字符，原生字符串可以包含任意行和文字



转义字符串(escaped string)由一对双引号界定  “内容”

```kotlin
val s = "Hello, world!\n"
```



原生字符串(raw string)由三个双引号界定 """内容"""

```kotlin
val text = """
    for (c in "foo")
        print(c)
"""
```



### 字符串模板

在Kotlin中字符串模板用$美元符来表示

```kotlin
val i = 10
val s = "i = $i" // evaluates to "i = 10"
val s = "abc"
val str = "$s.length is ${s.length}" // evaluates to "abc.length is 3"
```

