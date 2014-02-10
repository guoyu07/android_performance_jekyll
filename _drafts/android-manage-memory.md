---
layout: post
title:  "Android中的内存管理"
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
为了满足内存操作的需要，Android会尝试在进程之间共享内存，这个过程会按照如下的方式进行:

* 每一个App进程都派生(fork)自一个叫做“Zygote”的进程。“Zygote”进程在系统启动时就被创建了，它加载了一些公共的框架和资源（比如Activity的主题样式）。为了启动一个新的App进程，系统会从“Zygote”的进程派生出一个新进程，然后再新的进程中加载并运行App的代码。这个过程允许大多数分配给框架和资源的内存页在所有App进程中共享。

* 大多数静态数据都被内存映射到一个进程中。这可以使相同的数据在不同进程中共享，也可以按需要换出（page out）。比如静态数据包括：Dalvik虚拟机的代码（在可以直接进行内存映射的预先连接好的.odex文件中的），app资源以及一些传统项目元素，比如在.so文件中的本地代码。

* 在很多情况下，Android通过显示方式在多个进程中共享相同的内存区域。例如，surface在app和屏幕合成器（screen compositor）之间使用共享内存，游标缓冲区（cursor buffers）在 content provider 和 客户端app程序之间使用共享内存。

由于在Android中广泛使用共享内存，所以在开发过程中你要更加留意你的app使用了多少内存。想了解你的app究竟使用了多少内存，可以参考[Investigating Your RAM Usage](http://developer.android.com/tools/debugging/debugging-memory.html)。

***
##分配和回收App的内存
下面是Android针对的App进行内存分配和回收的关键点：

* 每个进程的Dalvik堆空间被限制在一个虚拟的内存区域中。其定义了堆空间的逻辑大小，它会按需增长（但是会有个上限，这个上限是系统为每个app定义的）。

* 这个堆空间的逻辑大小值和堆使用的物理内存大小是不一样的。当你查看你的App的堆内存的使用情况时，Android会计算出一个叫做PPS（Proportional Set Size）的值，这个值计算的是和和其他进程共享的那部分内存——但是这个值只时按占用比例来计算的，和有多少个App共享这块内存有关系（译者注：比如三个进程共享一个类库，这个类库占用30页的内存，那么每个进程针对这个类库的PSS值时10页内存的量）。PSS的总数可以反映你的物理内存的使用情况。更过关于PSS的解释，可以参考 [Investigating Your RAM Usage](http://developer.android.com/tools/debugging/debugging-memory.html#ViewingAllocations)。

* Dalvik不会压缩堆的逻辑空间大小，这意味着Android不会将堆整理成紧凑的空间。只有当堆尾有不使用的空间时Android才会缩小这个逻辑堆空间的大小。但是这不意味着堆使用的物理内存不能被回收。在垃圾回收之后，Dalvik会遍历堆空间并找出没用使用的内存页，然后通过`madvise`将这些页返回给内核。所以，大块区域的分配和释放会导致回收所有（或大部分）使用过的物理内存。但是，回收小块区域可能没那么有效，因为小块区域的内存页可能仍然被某些没有释放的东西共享着。

***
##限制App的内存

为了维持一个多任务的环境，Android把每个App能够使用的堆内存大小限制死了。确切的堆内存大小限制会根据设备总的内存大小不同而不同。如果你超过了这个限制，就会抛出 OutOfMemoryError 异常。

某些情况下，你可能想知道在你的设备上真正可以使用的堆内存到底有多少——比如，你想知道在内存缓存中放多少数据是安全的。你可以通过调用`getMemoryClass()`来获得这个数据，该方法会返回一个整数，这个整数代表你的堆空间还有多少兆字节的空间可用。下文中我们会讨论这个问题，在《检查你应该使用多少内存》一节。

***
##App的切换

Android在进行App切换时，没有使用交换空间，而是把那些不在前端展示（用户不可见）的应用组件放到一个LRU（最近最少使用）的缓存中。举个例子，当你第一次启动一个应用时，系统会为其创建一个进程，当用户离开这个应用时，它的进程并不退出。系统会把这个进程缓存住。所以，当用户会返回这个App时，这个缓存的进程会被重用，这样App的切换就会很快。

如果你的App的进程被缓存了，而且其持有当前并不适用的内存，那么它会限制系统整体的性能。所以，当系统内存吃紧时，系统会依据LRU原则干掉缓存中最近最少使用的进程，但也会酌情考虑到那些内存密集型的进程。如果想让你的进程在缓存中待更长的时间，请参考下面关于如何释放引用的章节。

关于Android如何缓存进程以及如何决定杀死某个进程的更多信息，请参考 [Processes and Threads ](http://developer.android.com/guide/components/processes-and-threads.html)。

***
#你应该如何管理内存

在你开发App的各个阶段你都应该考虑到内存限制的问题，包括在设计阶段。这里有很多有效的做法，把这些方法整合起来，举一反三会使你的App更高效。

在设计和正式开发你的App时，应用下面的技术会使你更有效地使用内存。

##保守地使用服务

如果需要在后台启一个服务，那么不要让它一直在运行，除非这个服务确实有任务在执行。而且，在后台任务完成后，需要停止后台服务时要更加小心，不要让服务关闭失败。

当你启动一个服务时，系统一般会保持住它的进程。这使得这个服务进程开销很大，因为它所占用的内存不会被其他程序使用，也不能被换出。这回降低系统在LRU缓存中能够保持的线程数，会使app切换变得很低效。当内存吃紧时也会造成系统的不稳定，因为系统可能不会为所有服务维持住足够的进程。

限制服务寿命的最好方法是使用[IntentService](http://developer.android.com/reference/android/app/IntentService.html),主要因为当捕获到某个Intent时就会启动，任务执行结束也会自杀。更多信息请阅读[Running in a Background Service ](http://developer.android.com/training/run-background-service/index.html)。

让你不再需要服务仍然处于运行状态是安卓内存管理中**最最严重的错误**。所以，不要贪图让你的服务一直运行着，这不仅不会增加你的App表现欠佳的风险，用户最终也会发现这些不良行为，然后把你的App卸载之。

***
##当你的用户界面隐藏时记得释放内存

当用户切换到另外一个App时，你的 UI 会被隐藏起来，这时记得释放掉那些只有你自己的 UI 才用到的资源。此时释放 UI 资源可以显著地系统可以缓存的进程数，这对用户体验的质量有直接影响。

通过在你的 Activity 实现[onTrimMemory() ](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#onTrimMemory(int))回调方法，当用户退出你的 UI 时，你会收到通知。你应该用这个方法监听 [TRIM\_MEMORY\_UI\_HIDDEN](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_UI_HIDDEN)事件，它表示你的 UI 现在已经隐藏掉了，你需要释放掉你的 UI 独占的那些资源。

注意，只有当你 App 进程的虽有UI组件都被隐藏时，你才能在 [onTrimMemory() ](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#onTrimMemory(int\))中收到[TRIM\_MEMORY\_UI\_HIDDEN](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_UI_HIDDEN)事件。这和[onStop()](http://developer.android.com/reference/android/app/Activity.html#onStop(\))回调函数不同，后者是当Activity实例被隐藏时才会调用，它经常会在你的App中从一个Activity切换到另外一个Activity中时发生。所以，虽然你需要在[onStop()](http://developer.android.com/reference/android/app/Activity.html#onStop(\))方法中释放一些Activity的资源，比如网络连接，或者注销掉广播receivers，但是通常你在收到[onTrimMemory(TRIM\_MEMORY\_UI\_HIDDEN)](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#onTrimMemory(int\))之前不应该释放掉UI资源。这可以确保在你的App中用户可以很快地进行Activity的切换。

##内存紧张时释放内存

在你的应用生命周期的各个阶段，当设备的整体内存使用率逐渐降低时，你也会通过[onTrimMemory()](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#onTrimMemory(int\))得到通知。当收到下面的事件时，你需要进一步地释放资源：

* [TRIM\_MEMORY\_RUNNING\_MODERATE](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_RUNNING_MODERATE)
<br/>
这时你的App不用担心会被杀死，但是此时设备内存已经开始吃紧，系统准备要从LRU缓存中干掉一些进程了。

* [TRIM\_MEMORY\_RUNNING\_LOW](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_RUNNING_LOW)
<br/>
这时你的App也不用担心会被杀死，但是此时设备内存更加紧张，所以你应该释放掉一些不用的资源来改善系统性能了（这也会影响直接你的App的性能）。

* [TRIM\_MEMORY\_RUNNING\_CRITICAL](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_RUNNING_CRITICAL)
<br/>
这时你的App还处于运行状态，但是系统已经准备好要给你掉LRU缓存中大部分进程了，所以你也危险了，你最好释放掉一些非关键的资源。如果系统不能回收到足够多的资源，那么系统将会开始杀死LRU缓冲中的而所有进程，包括哪些系统倾向于保持住的进程，比如后台服务进程。

如果你的App进程在缓存中，也能回从[onTrimMemory()](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#onTrimMemory(int\))方法中收到如下消息事件:

* [TRIM\_MEMORY\_BACKGROUND](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_BACKGROUND)
<br/>
此时，系统中内存吃紧，你的进程离LRU缓存队列的开始位置还比较近。虽然你的App进程被干掉的风险还不大，但是系统已经开始杀死LRU队列中的进程了。你应该释放一些可以很快恢复的资源，这样你的进程可以继续留在队列里，同时，当用户返回你的App是，也可以很快恢复过来。

* [TRIM\_MEMORY\_MODERATE](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_MODERATE)
<br/>
此时，系统中内存吃紧，你的进程已经接近LRU缓存队列的中部了。如果内存进一步紧张，你的进程很可能被干掉。

* [TRIM\_MEMORY\_COMPLETE](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_COMPLETE)
<br/>
此时，系统中内存吃紧，而且如果系统不能立即获得足够的内存，那么你的进程会成为下一个被杀死的进程。你应该释放所有非关键的资源来保持你app的状态。

因为[TRIM\_MEMORY\_COMPLETE](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_COMPLETE)是在 API 14 版本中添加进来的，低版本中你可以使用 [onLowMemory()](http://developer.android.com/reference/android/content/ComponentCallbacks.html#onLowMemory(\))回调方法，它基本上等价于[TRIM\_MEMORY\_COMPLETE](http://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_COMPLETE)事件。

> **注意：**当系统开始杀死LRU缓存中的进程中时，虽然时按照自下而上进行的，但是系统也会考虑干掉那些占用更多内存的进程，因为干掉它所回收的内存也就更多。所以，当你的进程在LRU缓存中时，你所占用的内存越少，进程的生存几率就越高。

***
##检查你应该使用多少内存
正如前文中我们提到的，由于每种Android设备上系统可用物理内存大小的总量是不一样的，所以对堆空间大小的限制也是不一样的。通过[getMemoryClass()](http://developer.android.com/reference/android/app/ActivityManager.html#getMemoryClass(\))可以获得你有多少可用的堆空间（这是个估算值），以兆字节为单位。如果你分配的内存超过了这个限制，就会抛出 `OutOfMemoryError` 异常。

在非常特殊的情况下，你可以在mainifest文件中设置 [largeHeap](http://developer.android.com/guide/topics/manifest/application-element.html#largeHeap)属性来获得更大的堆空间，这时，你可以通过[getLargeMemoryClass()](http://developer.android.com/reference/android/app/ActivityManager.html#getLargeMemoryClass(\)) 方法获可用的堆空间估算值。


但是，这个可以活的更大对内存的能力只适用于那些确实需要更多内存的应用（比如，用来编辑照片的App）。不到万不得已不要使用这个特性，这很容易早曾内存溢出，除非你能清楚地知道你的内存是什么时候被分配的，以及为什么持有这块内存。但是，及时你有自信运用这个特性，我们建议你还是尽可能的避免。占用更多的内存会系统整体性能造成伤害，任务切换时会变得更慢，垃圾回收时的时间也会更长。

另外，这个申请更大堆空间的特性也是很据设备不同而不同，在某些限制内存使用的设备上，大堆空间大小和常规空间的大小是一样的。所以，及时你使用了这个特性，你也应该只使用[getMemoryClass()](http://developer.android.com/reference/android/app/ActivityManager.html#getMemoryClass(\))方法来检查可用的堆空间，并努力将内存维持在这个限制之下。

***
##避免在 bitmap 上浪费内存
当你加载一个bitmap时，你需要显示多大分辨率的图就加载多大分辨率的图，如果原始图片分辨率过大，就将其缩放。记住，bitmap的分辨率越大占用的内存就越多，因为图片的X轴和Y轴的尺寸更大了。

>**注意：**在Android 2.3.x(API 10) 及以下，bitmap对象再堆空间占用的内存是一样的，和分辨率无关（实际的像素数据单独保存在本地内存空间中的（native memory））。这使得针对 bitmap 内存分配的调试变得很困难，以为大多数的内存分析软件看不到本地内存空间。但是，在Android 3.0（API 11）以后，bitmap的像素数据分配在App的Dalvik堆空间上，这就改善了垃圾回收和可调式的能力。所以，如果你发现在老版本上调试内存问题很麻烦，那就切换到Android3.0或更高版本再调试。

更多关于bitmap的使用技巧，请参考 [Managing Bitmap Memory](http://developer.android.com/training/displaying-bitmaps/manage-memory.html)。

***
##使用优化过的数据容器

要利用好Android框架中的一些优化过的数据容器，比如 [SparseArray](http://developer.android.com/reference/android/util/SparseArray.html), [SparseBooleanArray](http://developer.android.com/reference/android/util/SparseBooleanArray.html), 和 [LongSparseArray](http://developer.android.com/reference/android/support/v4/util/LongSparseArray.html)。我们常用的 HashMap 的实现，在内存方面是比较低效的，因为每组映射都需要对象作为入口。另外，[SparseArray](http://developer.android.com/reference/android/util/SparseArray.html)更高效，是因为它避免了autobox操作（就是把原始类型提升为对象类型，比如把int类型提升为Integer类型）。也不好害怕使用数组类型，也可以酌情使用。

***
##保持对内存负载的铭感度？？？？

