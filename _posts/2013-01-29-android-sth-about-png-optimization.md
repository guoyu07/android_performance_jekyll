---
layout: post
title:  "Android中关于PNG优化的那点事"
date:   2014-01-29 17:32:46
categories: android
---

以前做Web开发时，在上线前我们会把png和jpg图片再压缩一遍，已保证文件最小。Android中会使用大量的png图片来绘制UI组件，由此我们也自然联想到是否可以通过同样的手段来优化Android中的PNG图片，进而减小apk文件的大小。但是经过实验证明，这种做法是徒劳的，因为在打包的过程中Android自带的aapt工具已经自动帮你做了这件事，而且目前我还没有发现如何可以改变这种方式（比如禁用自动优化，或者改变压缩比）。

><strong>Note:</strong> Image resources placed in res/drawable/ may be automatically optimized with lossless image compression by the aapt tool during the build process. For example, a true-color PNG that does not require more than 256 colors may be converted to an 8-bit PNG with a color palette. This will result in an image of equal quality but which requires less memory. So be aware that the image binaries placed in this directory can change during the build. If you plan on reading an image as a bit stream in order to convert it to a bitmap, put your images in the res/raw/ folder instead, where they will not be optimized.

>来自 ：[http://developer.android.com/guide/topics/graphics/2d-graphics.html#drawables](http://developer.android.com/guide/topics/graphics/2d-graphics.html#drawables)

***

下面我们通过实验来证明这个结论：

首先要强调一下，我们这里说的优化或这压缩指的是无损的（lossless），是不在不改变图片质量的前提下来减小图片的大小。

先简单介绍一下我们用到的图片优化工具 [OptiPNG](http://optipng.sourceforge.net/)。它的原理主要就是有效减少颜色数，合并相似颜色等等，比如你用真色彩模式创建的一张单色png图片，优化后会被转成PNG8的模式，具体可以参考这篇文章 [A guide to PNG optimization](http://optipng.sourceforge.net/pngtech/optipng.html)。

我们有两个样例图片：

<p>demo_before_opt.png(395k):</p>
<img title="demo_before_opt.png" src="/images/android-sth-about-png-optimization/demo_before_opt.png" width="200" />

<p>demo_bg_before_opt.png(12k):</p>
<img title="demo_bg_before_opt.png" src="/images/android-sth-about-png-optimization/demo_bg_before_opt.png"  />

我们通过 optipng 工具分别对两张图片进行优化：

<pre>
> optipng.exe -o7 demo_before_opt.png -out demo_after_opt.png

** Processing: demo_before_opt.png
609x607 pixels, 3x8 bits/pixel, RGB
Input IDAT size = 403529 bytes
Input file size = 404213 bytes

Trying:
  zc = 9  zm = 9  zs = 0  f = 2         IDAT size = 375122
  zc = 9  zm = 8  zs = 0  f = 2         IDAT size = 374910
  zc = 9  zm = 9  zs = 1  f = 2         IDAT size = 372276
  zc = 9  zm = 8  zs = 1  f = 2         IDAT size = 372269

Selecting parameters:
  zc = 9  zm = 8  zs = 1  f = 2         IDAT size = 372269

Output file: demo_after_opt.png

Output IDAT size = 372269 bytes (31260 bytes decrease)
Output file size = 372365 bytes (31848 bytes = 7.88% decrease)
</pre>

<pre>
> optipng.exe -o7 demo_bg_before_opt.png -out demo_bg_after_opt.png

** Processing: demo_bg_before_opt.png
100x100 pixels, 3x8 bits/pixel, RGB
Reducing image to 8 bits/pixel, 132 colors in palette
Input IDAT size = 12287 bytes
Input file size = 12381 bytes

Trying:
  zc = 9  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 9  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 8  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 8  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 7  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 7  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 6  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 6  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 5  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 5  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 4  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 4  zm = 8  zs = 0  f = 0         IDAT size = 7367
  zc = 2  zm = 9  zs = 0  f = 0         IDAT size = 7367
  zc = 1  zm = 9  zs = 0  f = 0         IDAT size = 7361
  zc = 9  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 9  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 8  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 8  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 7  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 7  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 6  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 6  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 5  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 5  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 4  zm = 9  zs = 1  f = 0         IDAT size = 6987
  zc = 4  zm = 8  zs = 1  f = 0         IDAT size = 6987
  zc = 1  zm = 9  zs = 2  f = 0         IDAT size = 6987
  zc = 1  zm = 8  zs = 2  f = 0         IDAT size = 6987

Selecting parameters:
  zc = 1  zm = 9  zs = 2  f = 0         IDAT size = 6987

Output file: demo_be_after_opt.png

Output IDAT size = 6987 bytes (5300 bytes decrease)
Output file size = 7489 bytes (4892 bytes = 39.51% decrease)

</pre>

命令中的 -o7 表示使用最优策略，但是时间较慢。更多 optipng 的用法参见 [optipng-0.7.4.man.pdf](http://optipng.sourceforge.net/optipng-0.7.4.man.pdf)

两张图片图片分别被压缩了 7.88% 和 39.51%，优化后的图片为：

<p>demo_after_opt.png(363k):</p>
<img title="demo_after_opt.png" src="/images/android-sth-about-png-optimization/demo_after_opt.png" width="200" />

<p>demo_bg_after_opt.png(7.31k):</p>
<img title="demo_bg_after_opt.png" src="/images/android-sth-about-png-optimization/demo_bg_after_opt.png"  />

而且优化后的 demo\_bg\_after\_opt.png 被转成了PNG8的格式，我们通过 [TweakPNG](http://entropymine.com/jason/tweakpng/) 看下优化前后两个png的文件头便知。

<p>demo_bg_before_opt.png 的头信息:</p>
<img title="demo_bg_before_opt.png 的头信息" src="/images/android-sth-about-png-optimization/demo_bg_before_opt_header.png"  />

<p>demo_bg_after_opt.png 的头信息:</p>
<img title="demo_bg_after_opt.png 的头信息" src="/images/android-sth-about-png-optimization/demo_bg_after_opt_header.png"  />

<p>现在我们有了4个样例文件：</p>
<img title="所有样例文件" src="/images/android-sth-about-png-optimization/build_before.png"  />

最后我们把这4个文件放到android工程中的drawable目录下，然后打程apk包。再把apk包中的res目录解压出来看看这几个文件有什么变化，如下图：

<img  src="/images/android-sth-about-png-optimization/result.png"  />

demo_before_opt.png 和 demo_bg_before_opt.png 都比原始图片要小，说明android的打包程序已经帮你了图片优化。但是，demo_after_opt.png 和 demo_bg_after_opt.png 要比前面优化过的图片要大，而且结果中demo_after_opt.png 和 demo_before_opt.png 的大小都一样，而且连md5值都一样， 说明自行优化png图片是不起作用的，android有自己的优化策略。

总结：
目前来看，android有自带的png优化策略，自行优化是徒劳的。