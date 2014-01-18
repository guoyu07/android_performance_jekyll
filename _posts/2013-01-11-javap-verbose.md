---
layout: post
title:  "读懂 javap -verbose"
date:   2014-01-11 18:02:01
categories: java
---


javap是jdk自带的一个工具，可以反编译class文件，是我们在做java代码性能分析时必不可少的一个工具。
我们先写个简单的代码，然后我们在逐个分析 javap 解析出来的内容。

{% highlight java linenos %}
public class TestJavap {

    public static int add(int a, int b) {
		int r = a + b;
		return r;
    }

    public static void main(String[] args) {
		int r = add(15, 16);
		System.out.println(r);
    }

}
{% endhighlight %}

执行 javap -v TestJavap 之后获得的内容如下：

<pre>
D:\workspace\test_java\bin>javap -v TestJavap.class
Classfile /D:/workspace/test_java/bin/TestJavap.class
  Last modified 2013-12-31; size 643 bytes
  MD5 checksum 03f49f751716ceb852c190bfb54cbb2f
  Compiled from "TestJavap.java"
public class TestJavap
  SourceFile: "TestJavap.java"
  minor version: 0
  major version: 50
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Class              #2             //  TestJavap
   #2 = Utf8               TestJavap
   #3 = Class              #4             //  java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Methodref          #3.#9          //  java/lang/Object."<init>":()V
   #9 = NameAndType        #5:#6          //  "<init>":()V
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               LTestJavap;
  #14 = Utf8               add
  #15 = Utf8               (II)I
  #16 = Utf8               a
  #17 = Utf8               I
  #18 = Utf8               b
  #19 = Utf8               r
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Methodref          #1.#23         //  TestJavap.add:(II)I
  #23 = NameAndType        #14:#15        //  add:(II)I
  #24 = Fieldref           #25.#27        //  java/lang/System.out:Ljava/io/PrintStream;
  #25 = Class              #26            //  java/lang/System
  #26 = Utf8               java/lang/System
  #27 = NameAndType        #28:#29        //  out:Ljava/io/PrintStream;
  #28 = Utf8               out
  #29 = Utf8               Ljava/io/PrintStream;
  #30 = Methodref          #31.#33        //  java/io/PrintStream.println:(I)V
  #31 = Class              #32            //  java/io/PrintStream
  #32 = Utf8               java/io/PrintStream
  #33 = NameAndType        #34:#35        //  println:(I)V
  #34 = Utf8               println
  #35 = Utf8               (I)V
  #36 = Utf8               args
  #37 = Utf8               [Ljava/lang/String;
  #38 = Utf8               SourceFile
  #39 = Utf8               TestJavap.java
{
  public TestJavap();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LTestJavap;

  public static int add(int, int);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: iload_2
         5: ireturn
      LineNumberTable:
        line 4: 0
        line 5: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0     a   I
            0       6     1     b   I
            4       2     2     r   I

  public static void main(java.lang.String[]);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: bipush        15
         2: bipush        16
         4: invokestatic  #22                 // Method add:(II)I
         7: istore_1
         8: getstatic     #24                 // Field java/lang/System.out:Ljava/io/PrintStream;
        11: iload_1
        12: invokevirtual #30                 // Method java/io/PrintStream.println:(I)V
        15: return
      LineNumberTable:
        line 9: 0
        line 10: 8
        line 11: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  args   [Ljava/lang/String;
            8       8     1     r   I
}
</pre>
很长很恐怖，是吧。。。（如果是一个实际项目的class文件，那会恐怖得令人发指），别急，让我们来一点一点地分析：

<pre>
Classfile /D:/workspace/test_java/bin/TestJavap.class
  Last modified 2013-12-31; size 643 bytes
  MD5 checksum 03f49f751716ceb852c190bfb54cbb2f
  Compiled from "TestJavap.java"
public class TestJavap
  SourceFile: "TestJavap.java"
  minor version: 0
  major version: 50
</pre>
这部分不用多说，大家一看就明白。主要就是记录一些基础的版本信息。minor version: 0 major version: 50 指的是这个class文件编译时所使用的jdk版本号。

<h3>常量池:</h3>

<pre>
Constant pool:
   #1 = Class              #2             //  TestJavap
   #2 = Utf8               TestJavap
   #3 = Class              #4             //  java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Methodref          #3.#9          //  java/lang/Object."<init>":()V
   #9 = NameAndType        #5:#6          //  "<init>":()V
   .......
</pre>
Constant Pool （常量池），在java虚拟机中是个重要的概念。我们可以这样理解一下，这个“池子”记录了java程序运行所需要的所有符号，包括变量名、方法名、类名、字符串等一切符号。在下面的介绍中你会看到，在java方法执行时会经常引用常量池中的内容。#1，#2 这样的数字可以理解为常量池中的每一项的“索引地址”，字节码指令会经常使用这个索引来引用对应的符号。

这里张图可以加深对常量池的理解：

(图1：java 虚拟机的数据结构)
<a href="/images/java_class_file_parts.png" target="_blank"><img src="/images/java_class_file_parts.png" /></a>

(图2：java class 文件结构)
<a href="/images/java_vm_data_structure.png" target="_blank"><img src="/images/java_vm_data_structure.png" /></a>


是不是觉得jvm运行时离不开常量池

更多关于常量池的介绍可以参考：[http://en.wikipedia.org/wiki/Java_class_file#The_constant_pool][link1], 以及《深入java虚机》一书



下面是重点，我们会详细介绍方法字节码表示的含义。
比如方法 add 对应的java代码和字节码表示为：

{% highlight java linenos %}
public static int add(int a, int b) {
	int r = a + b;
	return r;
}
{% endhighlight %}

<pre>
  ......
  public static int add(int, int);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: iload_2
         5: ireturn
      LineNumberTable:
        line 4: 0
        line 5: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       6     0     a   I
            0       6     1     b   I
            4       2     2     r   I
   ......
</pre>

其中 flags: ACC_PUBLIC, ACC_STATIC 这一行我觉得不用细讲，一看就明白，这是类或方法的访问标识，用来定义他们的访问权限的。还有 ACC_FINAL ACC_ABSTRACT 等。他们和 public 、static、final 、abstract 这些关键字是对应的。

<h3>局部变量表:</h3>
下面我们先来介绍 LocalVariableTable（局部变量表）

我们要先有记住一点，jvm是基于栈的运算，先看一下上面的图1（Java虚拟机运行时的数据结构）。每个java线程在运行时，jvm都会为其分配一个“栈空间”（就是一个内存区域），主要包括一个PC寄存器（记录当前线程运行的下一条指令），JVM栈空间，本地栈空间（本地代码，一般是C写的lib可以理解为JNI的方式调用的代码，和我们自己写的java代码无关了）。当某个java方法运行时，jvm会创建一个“栈帧”（也是一段内存空间），我们要介绍的LocalVariableTable就是“栈帧”的一部分，另外“栈帧”还包括我们常听说的“操作数栈（Operand Stack）”和对常量池的引用（Reference To Constant Pool）。局部变量表中记录了一个java方法运行时锁需要的局部变量名（Name 这一列）， Signature是类型描述符，I就表示int类型（更多类型描述符参见：[http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3][link2]）。

这个还要介绍一个“Slot”的概念，一个Slot就可以理解为一个32位（4字节）的内存单位。在我们的例子中，参数a、b临时变量r都是int类型，在java中，int类型就是一个4字节长度，即1个slot。在我们的例子中，LocalVariableTable中有三个变量，都是int类型，需要3个slot，所以看到locals =3 这一行 就应该明白是什么意思了吧。

我们在深入一点，把a，b和r都换成Long类型，在javap -v 一下，看看会变成什么样子：
代码：
{% highlight java linenos %}
public static long add(long a, long b) {
	long r = a + b;
	return r;
}
{% endhighlight %}
对应的字节码为：
<pre>
  .......
  public static long add(long, long);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=4, locals=6, args_size=2
         0: lload_0
         1: lload_2
         2: ladd
         3: lstore        4
         5: lload         4
         7: lreturn
      LineNumberTable:
        line 4: 0
        line 5: 5
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0     a   J
            0       8     2     b   J
            5       3     4     r   J
   .......
</pre>

是不是能找到点感觉啦? 因为在java中long类型是64位，8字节，要占用2个slot，所以3个变量共占用6个slot，所以这里 locals = 6。Slot这里一列也不一样了，是吧，说明，Slot这一列可以看作变量空间的入口索引位置（Signature下的J是long类型的类型描述符）。

<h3>栈宽:</h3>
stack 指的是栈的宽度——就是执行这个方法时，为这个方法的操作数栈定义多少个slot，注意，这个宽度足以容纳当前方法所有运算所需要的操作数，下面我们举例说明。
上面的例子中，只有一个 <code>a + b</code> 的操作，每个参数都是long型（即2个slot）, 执行这个加法运算的过程是这样的，lload_0 指令把 LocalVariableTable 中索引为 0 的操作数（变量a）压入操作数栈中，lload_2 把索引为 2 的操作数也压入栈中，注意，这里操作数栈中已经压入了两个 long 类型，共4个slot，然后 ladd 指令从栈中弹出这两个操作数（此时操作数栈空了），运算结束后在把运算结构再次压入栈中，此时操作数栈中只有一个long类型的数据（占用2个slot），然后 lstore 把栈中的结果保存在局部变量表中索引为4的位置（即变量r）。在这个过程中，“最多”占用4个slot（就是把a和b都压入栈中的时候），所以 <code>stack=4</code>。

<h3>字节码偏移位置:</h3>
Code 代码前的标号是字节码指令的偏移（java的字节码文件组织得是很紧凑的，每个字节都有其具体的含义）。
jvm 中每个字节码占用1个字节，上面了例子中，lload_0 、lload_2、ladd 3个指令由于没有操作数，所以它们几个的偏移量分别为0，1，2。第四个指令 lstore 后面跟了一个操作数索引参数（1个字节），其占用2个字节，所以下一个质量的偏移量是从 5开始，一次类推。更多java字节码质量参见：[Java bytecode instruction listings][link3]

<h3>LineNumberTable:</h3>
LineNumberTable 记录字节码行号和源代码行号的对应关系。
比如 line 4: 0，左边的4代表源码的行号，后边的0代表字节码的起始偏移地址。这个信息是用来调试用了，我们经常看到的java抛出的异常时锁所携带的线程调用栈的信息，就是跟这个表有关系。


[link1]:http://en.wikipedia.org/wiki/Java_class_file#The_constant_pool
[link2]:http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.3
[link3]:http://en.wikipedia.org/wiki/Java_bytecode_instruction_listings