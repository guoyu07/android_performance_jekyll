---
layout: post
title:  "App内存管理"
date:   2014-01-28 10:49:01
categories: android
---

本文内容翻译自：http://developer.android.com/training/articles/memory.html

随机存取存储器(RAM)再任何软件开发环境中都是宝贵的资源，但是在移动操作系统中，内存资源更为宝贵，使用时也会收到限制。虽然Android的Dalvik虚拟机有运行时的垃圾回收机制，但是这不意味着你的App可以随便使用内存。

为了让垃圾回收器回收内存，你得避免造成内存泄漏（通常是持有全局对象的引用造成的），并且在适当的时候释放`Reference`类型的对象（下文中会进一步讨论这个问题）。对于大多数App，Dalvik虚拟机的垃圾回收器会处理好剩下的内存回收：当对象离开当前活动线程的作用域时，系统会回收其内存空间。

本文主要介绍Android是如何处理和分配内存的，以及如何在开发App时如何主动地较少内存的使用。For more information about general practices to clean up your resources when programming in Java, refer to other books or online documentation about managing resource references。更多关于如何管理你的资源，你可以参考其他的书籍或在线文档。如果你想分析已有App的内存使用情况，你可以阅读 [Investigating Your RAM Usage](http://developer.android.com/tools/debugging/debugging-memory.html)。



