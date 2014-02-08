---
layout: post
title:  "App内存管理"
date:   2014-02-07 10:49:01
categories: android
---

本文内容翻译自：http://developer.android.com/training/articles/memory.html

随机存取存储器(RAM)再任何软件开发环境中都是宝贵的资源，但是在移动操作系统中，内存资源更为宝贵，使用时也会收到限制。虽然Android的Dalvik虚拟机有运行时的垃圾回收机制，但是这不意味着你的App可以随便使用内存。

为了让垃圾回收器回收内存，你得避免造成内存泄漏（通常是持有全局对象的引用造成的），并且在适当的时候释放`Reference`类型的对象（下文中会进一步讨论这个问题）。对于大多数App，Dalvik虚拟机的垃圾回收器会处理好剩下的内存回收：当对象离开当前活动线程的作用域时，系统会回收其内存空间。

本文主要介绍Android是如何处理和分配内存的，以及如何在开发App时如何主动地较少内存的使用。更多关于如何管理你的资源，你可以参考其他的书籍或在线文档。如果你想分析已有App的内存使用情况，你可以阅读 [Investigating Your RAM Usage](http://developer.android.com/tools/debugging/debugging-memory.html)。


***
# Android 如何管理内存
Android 不提供交换空间（swap space），但是它使用内存分页技术（[paging](http://en.wikipedia.org/wiki/Paging)）和内存映射技术（ [memory-mapping](http://en.wikipedia.org/wiki/Memory-mapped_files)，也称作 “mmapping”，译者注：主要就用于提高大文件的读写效率以及多进程之间的内存共享，也可参考[《内存映射文件原理探索 》](http://blog.csdn.net/mg0832058/article/details/5890688)）。这意味着你操作过的任何内存区域——不管是通过对象分配还是操作映射过的页——都会一直驻留在内存中，也不会被换出（译者注：我的理解是由于android没有提供交换空间，内存中的数据不会自动换入到磁盘中，这意味着我们能利用的只有物理内存）。所以，彻底释放内存的的唯一方法就是释放对象的引用，使垃圾回收器可以对其进行回收。但是这也伴随着一个问题：那些没用更改过但是已经被映射到内存中的文件，比如代码，如果系统需要征用它做占用的内存页时，它就会从内存中换出。

***
##共享内存
收拾收拾
