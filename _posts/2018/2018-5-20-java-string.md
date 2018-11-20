---
layout: post
title: 深入解析string
category: java
tags: [java]
---



### 概览
string被声明为final，因此他不可被继承

内部使用char数组存储数据，该数组被声明为final,这意味着value数组初始化之后就不能再引用其它的数组。并且String内部没有改变value数组的方法，因此可以保证String不可变。

```
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
```
### 不可变的好处

#### 1，可以缓存hash值
因为String的hash值经常被使用，例如string用做hashMap的key,不可变的特性使得hash也不可变，因此只需要进行一次计算

#### 2，String Pool的需要
如果一个String对象已经被创建过了，那么就会从String Pool中取得引用。只有String 是不可变的，才能使用String Pool
![](http://io.dbbaxbb.cn/assets/images/2018/java/1.png) <br/>
#### 3, 安全性
String经常作为参数，String不可变性可以保证参数不可变。例如在作为网络连接参数的情况下如果String是可变的，那么在网络连接过程中，String被改变，
改变String对象的那一方以为现在连接的是其它主机，而实际情况却不一定是。

#### 4, 线程安全
String不可变性天生具备线程安全，可以在多线程中安全的使用。

### String,StringBuffer 和StringBuilder

#### 1,可变性
* String不可变
* StringBuffer 和 StringBuilder

#### 2,线程安全
* String 不可变，因此是线程安全的
* StringBuilder 不是线程安全的
* StringBuffer 是线程安全的，内部使用 synchronized 进行同步

### String Pool
字符串常量池，保存着所有的字符串字面量，这些字面量在编译时期就确定，不仅如此，还可以使用String的intern()方法在运行过程中将字符串添加到常量池中。

当一个字符串调用intern()方法时，如果String Pool中已经存在一个字符串和改字符串相等时，那么就会返回String Pool中字符串的引用；否则就会在String Pool中添加一个新的字符串，并返回这个字符串的引用。

下面示例中，s1和s2采用new String()的方式新建两个不同的字符串，而S3好S4是用过s1.intern()方法取得一个字符串的引用。intern()首先把s1引用的字符串放到String Poll中，然后返回这个字符串的引用。因此S3和S4是同一个字符串
``` 
String s1 = new String("aaa");
String s2 = new String("aaa");
System.out.println(s1 == s2);           // false
String s3 = s1.intern();
String s4 = s1.intern();
System.out.println(s3 == s4);           // true
``` 
如果是采用“bbb”这种字面量的形式创建字符串，会自动将字符串放到String Pool中。
```
String s5 = "bbb";
String s6 = "bbb";
System.out.println(s5 == s6);  // true
```

在java 7之前，String Pool被放在运行时常量池中，它属于永久代，而在java7之后，String Pool被移到堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致OutOfMemoryError 错误。

### new String("abc")
使用这种方式一共创建的两个字符串对象（前提是String Pool中还没有'abc'字符串对象）
* “abc”属于字符串字面量，因此在编译时会在String Pool中创建一个字符串对象，指向这个“abc”的字符串面量
* 而使用new的方式会在堆中创建一个字符串对象

创建一个测试类，其main方法中使用这种方式来创建对象
```
public class NewStringTest {
    public static void main(String[] args) {
        String s = new String("abc");
    }
}
```
使用javap -verbose 进行反编译，得到以下内容
```
// ...
Constant pool:
// ...
   #2 = Class              #18            // java/lang/String
   #3 = String             #19            // abc
// ...
  #18 = Utf8               java/lang/String
  #19 = Utf8               abc
// ...

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/String
         3: dup
         4: ldc           #3                  // String abc
         6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
// ...
```
在 Constant Pool 中，#19 存储这字符串字面量 "abc"，#3 是 String Pool 的字符串对象，它指向 #19 这个字符串字面量。在 main 方法中，0: 行使用 new #2 在堆中创建一个字符串对象，并且使用 ldc #3 将 String Pool 中的字符串对象作为 String 构造函数的参数。

以下是String构造函数的源码，可以看到，在将一个字符串对象作为另一个字符串对像的构造函数参数时，并不会完全复制value数组的内容，而是都会指向同一个value数组
```

public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```
[参考链接](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20%E5%9F%BA%E7%A1%80.md)