

# Java String 总结

[TOC]

在java中String是一个不可变对象。不可变类有一个缺点，对于每个不同的值，都需要生成一个新的对象。但同时又由于不可变类其域不可变，可以通过非final的域延迟加载并缓存hashCode。

## 从源码中了解String

##### String的内部实现

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {    
	/** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
  //省略String类中的方法...
}
```

String对字符串的存储经由内部一个final修饰的char数组，String内部提供会造成字符串修改的方法，无一例外的返回一个新的String对象。

##### String.valueOf与Interger.toString的区别

```
 public static String valueOf(int i) {
        return Integer.toString(i);
    }
```

这两者没用任何区别，String.valueOf内部实现还是Integer.toString。其他基本类型同理。

## +运算符的重载

##### +运算符的使用

```java
String str1 = "a";
String str2 = str1 + "b";
```

以上代码在编译后

```
   L0
    LINENUMBER 5 L0
    LDC "a"   //创建一个内容为"a"的字符串对象，并将其引用存入常量池
    ASTORE 1 
   L1
    LINENUMBER 6 L1
    NEW java/lang/StringBuilder
    DUP
    ALOAD 1
    INVOKESTATIC java/lang/String.valueOf (Ljava/lang/Object;)Ljava/lang/String;
    INVOKESPECIAL java/lang/StringBuilder.<init> (Ljava/lang/String;)V
    LDC "b"
    INVOKEVIRTUAL java/lang/StringBuilder.append (Ljava/lang/String;)Ljava/lang/StringBuilder;
    INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
    ASTORE 2
```

可以看出，当有字符串对象参与+运算，会new出一个StringBuilder对象，接着调用append方法来不段添加字符。在这个过程中只生成了一个对象。

##### 在循环中使用+运算符

```java
String a = "";
for(int i = 0;i < 10;i++) {
		a += i;
}
```

```
  L0
    LINENUMBER 5 L0
    LDC ""
    ASTORE 1
   L1
    LINENUMBER 6 L1
    ICONST_0
    ISTORE 2
   L2
    GOTO L3
   L4  //循环体内部的代码
    LINENUMBER 7 L4
   FRAME APPEND [java/lang/String I]
    NEW java/lang/StringBuilder
    DUP
    ALOAD 1
    INVOKESTATIC java/lang/String.valueOf (Ljava/lang/Object;)Ljava/lang/String;
    INVOKESPECIAL java/lang/StringBuilder.<init> (Ljava/lang/String;)V
    ILOAD 2
    INVOKEVIRTUAL java/lang/StringBuilder.append (I)Ljava/lang/StringBuilder;
    INVOKEVIRTUAL java/lang/StringBuilder.toString ()Ljava/lang/String;
    ASTORE 1
```

重点看循环体内部的代码，可以看到每次进入循环后多会new一个StringBuilder对象，会造成过多的开销。这也是不推荐在循环中生成字符串使用+号的原因。

## switch对String的支持

代码就直接从网上找的

```java
public class switchDemoString {
     public static void main(String[] args) {
         String str = "world";
         switch (str) {
         case "hello": 
              System.out.println("hello");
              break;
         case "world":
             System.out.println("world");
             break;
         default: break;
       }
    }
}
```

对编译后的文件进行反编译

```java
public static void main(String args[]) {
       String str = "world";
       String s;
       switch((s = str).hashCode()) {
          case 99162322:
               if(s.equals("hello"))
                   System.out.println("hello");
               break;
          case 113318802:
               if(s.equals("world"))
                   System.out.println("world");
               break;
          default: break;
       }
  }
```

可以看出swtich本质上还是对int进行支持，但为了避免哈希值碰撞的问题，还会通过equals方法进行一次安全检查

## 字符串常量池

主要使用方法：

- 直接使用字面量（双引号）声明出的String对象会存入常量池。


- 不是字面量声明的String对象，可以使用String提供的intern方法。该方法会从字符串常量池中检查是否含有相同内容的对象，有则返回，没有则将该字符串对象加入常量池并返回。

字符串常量池中存放的都是对象的引用，在java中对象都存放在堆内存中。

当代码中出现<u>字面量形式创建字符串</u>时，JVM首先会检查字符串常量池中是否含有相同内容的字符串对象，如果有则<u>返回该对象的*引用*</u>。如果没有则创建新的字符串对象，然后将该对象的引用放入常量池，并返回该对象引用。

接下来分析这样一段代码

```java
String a = "java";
String b = "java";
System.out.println((a == b));
```

```java
true
```

声明并赋值变量a时，JVM首先检查常量池中是否有内容为java的对象存在（此处假设没有），那么会创建一个内容为java的对象，并存入常量池，然后返回该对象的引用。接着声明并赋值变量b，JVM在字符串常量池中发现有内容为java的对象存在，于是返回该对象的引用。



注：在JDK7版本中对常量池做了修改，所以JDK6与JDK7的代码运行结果可能有所不同。

- 将字符串常量池从Perm区移到了Java Heap区

## 实战解析

题目1

以下代码创建了几个对象

```java
String a = new String("a")
```

该代码创建了两个对象，第一个为存在字符串常量池中的字符串对象，第二个在Java Heap中的String对象

题目2

```java
    String a = "hello2"; 
    final String b = "hello";
    String d = "hello";
    String c = b + 2; 
    String e = d + 2;
    System.out.println((a == c));
    System.out.println((a == e));
```

```Java
true
false
```

当final变量是基本数据类型时，且其变量值在编译期间能确切知道，就会被当做编译时常量(constant variable)。其访问会按照Java语言对常量表达式的规定而做常量折叠。

题目3

```java
  public static void main(String[] args)  {
        String a = "hello2"; 
        final String b = getHello();
        String c = b + 2; 
        System.out.println((a == c));
 
    }
     
    public static String getHello() {
        return "hello";
    }
```

```java
false
```

由于在编译期间无法确切知道变量b的值，所以编译时没进行常量折叠，最终由于+运算符的重载生成新的String对象c

题目4

```java
	String d = "d";	
	final String a  = "a" + d;
	final String b = "b";
	String c = a + b;
	System.out.println((c=="ab"));
```

```java
false
```

题目5

```java
	final String a  = "a";
	final String b = "b";
	String c = a + b;
	System.out.println((c=="ab"));
```
```java
true
```

参考资料：[《Java String源码浅析》](http://blog.csdn.net/samjustin1/article/details/52253743) 

[《学会阅读Java字节码》](http://blog.csdn.net/dc_726/article/details/7944154 )

[《Java总结篇系列：Java String》](http://www.cnblogs.com/lwbqqyumidi/p/4060845.html)

[《java中的字符串常量池》](http://droidyue.com/blog/2014/12/21/string-literal-pool-in-java/)

[《深入解析Strin#intern》](http://tech.meituan.com/in_depth_understanding_string_intern.html)

