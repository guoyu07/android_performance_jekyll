---
layout: post
title:  "Java中static、private、public 方法哪个更快[draft]"
date:   2014-01-17 09:24:01
categories: java
---

在java中，同样的方法被声明不通的类型在访问速度上会有不同吗？如果不通会有多大差异？让我们功过实验来证明这一切。

我们有下面三段代码，运算逻辑相同，我们分别用static, private, public 来声明，然后分别对他们的运行时间：

{% highlight java  %}
public class TestStatic {

    static long add(long a, long b) {
		return a + b;
    }

    public static void main(String[] args) {
		long start = System.currentTimeMillis();
		for (long i = 0; i < 9999999999L; i++) {
			add(i,i+1);
		}
		System.out.println(System.currentTimeMillis() - start);
    }
}

{% endhighlight %}


{% highlight java  %}
public class TestPrivate {

    private long add(long a, long b) {
		return a + b;
    }

    public static void main(String[] args) {
		TestPrivate obj = new TestPrivate();
		long start = System.currentTimeMillis();
		for (long i = 0; i < 9999999999L; i++) {
			obj.add(i,i+1);
		}
		System.out.println(System.currentTimeMillis() - start);
    }
}

{% endhighlight %}


{% highlight java  %}
public class TestPublic {

    public long add(long a, long b) {
		return a + b;
    }

    public static void main(String[] args) {
		TestPublic obj = new TestPublic();
		long start = System.currentTimeMillis();
		for (long i = 0; i < 9999999999L; i++) {
			obj.add(i,i+1);
		}
		System.out.println(System.currentTimeMillis() - start);
    }
}
{% endhighlight %}

          
<br/>
表1：各方法执行5次所花时间的对比结果（单位毫秒）运行环境是早我的笔记本上（Dell E6410 上）

<table class="table table-bordered table-striped">
	<tbody>
		<tr>
			<th>#</th>
			<th>static 方法</th>
			<th>private 方法</th>
			<th>public 方法</th>
		</tr>
		<tr>
			<td>1</td>
			<td>16804</td>
			<td>20424</td>
			<td>20428</td>
		</tr>
		<tr>
			<td>2</td>
			<td>17061</td>
			<td>20291</td>
			<td>20246</td>
		</tr>	
		<tr>
			<td>3</td>
			<td>17044</td>
			<td>20629</td>
			<td>20604</td>
		</tr>	
		<tr>
			<td>4</td>
			<td>17064</td>
			<td>20207</td>
			<td>21107</td>
		</tr>
		<tr>
			<td>5</td>
			<td>16869</td>
			<td>20079</td>
			<td>20405</td>
		</tr>
	</tbody>	
</table>

从结果中可见，static 方法比 private 和 public 方法要快 15% 左右，private 和 public 消耗相差无几。

通过 javap -v 获得的字节码我们看到，在调用这几个方发的时候，jvm 使用了不同的指令：

static 实现中 main 方法的部分字节码：
<pre>
...
         6: goto          21
         9: lload_3
        10: lload_3
        11: lconst_1
        12: ladd
        13: **invokestatic**  #27                 // Method add:(JJ)J
        16: pop2
        17: lload_3
        18: lconst_1
        19: ladd
        20: lstore_3
        21: lload_3
...
</pre>

private 实现中 main 方法的部分字节码：
<pre>
...
       15: goto          35
       18: aload_1
       19: lload         4
       21: lload         4
       23: lconst_1
       24: ladd
       25: **invokespecial** #28                 // Method add:(JJ)J
       28: pop2
       29: lload         4
       31: lconst_1
       32: ladd
       33: lstore        4
       35: lload         4
...
</pre>

public 实现中 main 方法的部分字节码：
<pre>
...
        15: goto          35
        18: aload_1
        19: lload         4
        21: lload         4
        23: lconst_1
        24: ladd
        25: __invokevirtual__ #28                 // Method add:(JJ)J
        28: pop2
        29: lload         4
        31: lconst_1
        32: ladd
        33: lstore        4
        35: lload         4
...
</pre>

在看一下几种实现方式的add方法的字节码：

static 实现中 add 方法的字节码：
<pre>
  static long add(long, long);
    flags: ACC_STATIC
    Code:
      stack=4, locals=4, args_size=2
         0: lload_0
         1: lload_2
         2: ladd
         3: lreturn
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0     a   J
            0       4     2     b   J
</pre>

private 实现中 add 方法的部分字节码（需要用javap -v -p）：
<pre>
  private long add(long, long);
    flags: ACC_PRIVATE
    Code:
      stack=4, locals=5, args_size=3
         0: lload_1
         1: lload_3
         2: ladd
         3: lreturn
      LineNumberTable:
        line 8: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   LTestPrivate;
            0       4     1     a   J
            0       4     3     b   J
</pre>

public 实现中 add 方法的部分字节码：
<pre>
  public long add(long, long);
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=5, args_size=3
         0: lload_1
         1: lload_3
         2: ladd
         3: lreturn
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  this   LTestPublic;
            0       4     1     a   J
            0       4     3     b   J
</pre>

可以看到几个 add 方法字节码（Code部分）的实现几乎是一样的，而在调用这几个方法时jvm使用了 `invokestatic`，`invokespecial` 和 `invokevirtual` 三种不同的虚拟机指令。表1中的性能差异主要就是由这几条指令的操作方式所决定的，invokestatic 指令是基于**方法**（在编译时就知道该调用哪个方法）的指令，在进行栈帧切换（可以理解方法切换）时只需要把方法的参数入栈即可，从 “static 实现中 add 方法的字节码” 中我们可以看到其 LocalVariableTable（局部变量表）中只有a、b两个值。而 invokespecial 和 invokevirtual 是基于**实例**的指令，他们处理把a、b两个参数入栈之外，还要把对实例的引用（this指针）也同时入栈，所以在private和public实现方式的add方法字节码中，LocalVariableTable 还包括了this指针。所以这一点点额外的操作就决定了他们的性能差别。

>更多关于字节码的解释可参考之前的一篇文章 [《读懂 javap -verbose 》](/java/2014/01/11/javap-verbose.html)    
  
>关于 invokestatic 、invokespecial、invokevirtual 这几个指令的详解，可参考 [http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html][link1]


invokespecial 和 invokevirtual 在上面的例子并没有体现出明显的差别。我们再举一个例子比较一下，在这里例子中我们引入多态特性。我们把 TestPublic 改造一下，代码如下：

{% highlight java  %}

class TestBaseClass{
    public long add(long a, long b) {
		return a + b;
    }   
}

public class TestPublic extends TestBaseClass{
    
    public long add(long a, long b) {
		return a + b;
    }

    public static void main(String[] args) {
		TestBaseClass obj = new TestPublic();
		long start = System.currentTimeMillis();
		for (long i = 0; i < 9999999999L; i++) {
			obj.add(i,i+1);
		}
		System.out.println(System.currentTimeMillis() - start);
    }
}

{% endhighlight %}

我们在执行5次，看看结果是怎样的（单位毫秒）？

<table class="table table-bordered table-striped">
	<tbody>
		<tr>
			<th>1</th>
			<th>2</th>
			<th>3</th>
			<th>4</th>
			<th>5</th>
		</tr>
		<tr>
			<td>79712</td>
			<td>80419</td>
			<td>81648</td>
			<td>89341</td>
			<td>83449</td>
		</tr>
	</tbody>	
</table>

在 public 多态的情况下，同样的逻辑，花的时间是之前的4倍左右。这是由于 invokevirtual 指令属于“动态绑定”——即运行时才知道方法的所属类型是哪个，相对于动态绑定的是“静态绑定”——即编译时就知道要执行的方法属于哪个类。动态绑定不仅需要查方法表，而需要在运行时确定要引用的方法所属的类到底是哪个，这两中操作是比较耗时的。

> 关于动态绑定、静态绑定的内容可参考 [What is Static and Dynamic binding in Java with Example][link2]

> 关于 invokevirtual 指令如何确定动态绑定的类型可参考[http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokevirtual][link3]

###小结
本文锁讲述的内容并不会对你的实际项目有多大的性能提升，但是却可以指导我们养成一个“好”的编码习惯。对于独立的逻辑优先使用static 方式或者是private方式，没有必要的情况下，少用public方法，尤其在多态的模式下，public方法会有比较大的性能损耗。
因为在 java 中 invokestatic 、invokespecial 都属于静态绑定，其他的静态绑定还有声明为 final 的方法，他们在编译时就知道方法属于那个类，所以在运行时会比较快地定位到方法在内存中对应的字节码地址（在方法区中），不像动态绑定，还需要明确方法所在的类型并搜索方法表才能定位到。



[link1]:http://en.wikipedia.org/wiki/Java_class_file#The_constant_pool
[link2]:http://javarevisited.blogspot.com/2012/03/what-is-static-and-dynamic-binding-in.html
[link3]:http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.invokevirtual

[link_javap]:http://android-performance.com/java/2014/01/11/javap-verbose.html