---
layout: post
title:  "CSS transform 的基本原理(3)"
date:   2017-04-15 22:13:00
category: coding
---

[CSS transform 的基本原理(1)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%86.html)
[CSS transform 的基本原理(2)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%862.html)

## transform-origin

其实除了 transform 之外，还有一个 CSS 属性会对变换产生影响，那就是 transform-origin。
这个属性用于指定变换的原点。

考虑下面的两个例子

{% highlight css %}
.div1 {
    transform: scale(0.5, 0.5);
}
.div2 {
    transform-origin: left bottom;
    transform: scale(0.5, 0.5);
}
{% endhighlight %}

具体的区别大概是这样

![transform\_origin demo](http://crazydogs.github.io/images/css_transform_origin.png)

其中灰色的是变换前的正方形，蓝色的是 div1，红色的是 div2。也可以直接看这个
[demo](http://crazydogs.github.io/staticpage/css_transform_demo2.html)

改变原点，并不是一个线性变换操作，所以没有放在 transform 属性中也正常。但是改变原点，
有没有办法，也融入到 transform 中呢，就像之前说到的 translate 一样，通过扩展矩阵的阶数，
也变成了线性变换。

回想起来，translate 操作和 transform-origin 的区别是什么呢，就形式来说，好像都是平移呢，
如果我们把上面 demo 中的蓝色正方形向左下平移，是不是就可以达到红色正方形一样的效果了呢。

## 画一个表

考虑这样一个例子吧，我们须要用 CSS 来画一个时钟的钟面，或者说手表的表盘(圆形的)，
要求数字可以在阿拉伯数字和罗马数字间自由切换。如果说把钟面背景和数字一起做成一张背景图，
当然是最简单的了。但这样子就须要两张图了，一张罗马数字一张阿拉伯数字。

为了节省流量，我们决定，文字使用文本来实现。那，这 12 段文字，应该如何定位呢？
如果说直接使用绝对定位，当然也是可以的，但当表盘尺寸变化的时候，就要重新计算位置了，
明显不够优雅。

果然，使用 transform 进行旋转，才是正道。只要使用 transform-origin
指定旋转变换的原点到表盘的中心，然后每个数字依次旋转 30° 就可以了。关键代码如下

{% highlight html %}
<style>
    #clock {
        width: 400px;
        height: 400px;
        border: 2px solid #ddd;
        border-radius: 50%;
        position: relative;
    }
    .clock-number {
        position: absolute;
        width: 100%;
        height: 40px;
        font-size: 30px;
        text-align: center;
        transform-origin: center 200px;
    }
    .clock-number:nth-child(2) {
        transform: rotate(30deg);
    }
    .clock-number:nth-child(3) {
        transform: rotate(60deg);
    }
    ... 
</style>
<div id="clock">

    <div class="clock-number">12</div>
    <div class="clock-number">1</div>
    <div class="clock-number">2</div>
    ...
    <div class="clock-number">10</div>
    <div class="clock-number">11</div>
</div>
{% endhighlight %}

具体可以看 [demo 页面](http://crazydogs.github.io/staticpage/css_transform_clock_1.html)
效果大概是这样

![transform\_origin demo](http://crazydogs.github.io/images/css_transform_clock_1.png)

但是这样还没有达到我们的要求，我们期望的效果是这样


