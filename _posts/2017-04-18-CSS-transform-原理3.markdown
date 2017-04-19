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

![transform origin demo](http://crazydogs.github.io/images/css_transform_origin.png)

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

![transform origin demo1](http://crazydogs.github.io/images/css_transform_clock_1.png)

但是这样还没有达到我们的要求，如果是罗马数字的话还好，但阿拉伯数字通常文字都会保持向上的方向
，我们期望的效果是这样。

![transform origin demo2](http://crazydogs.github.io/images/css_transform_clock_2.png)

那其实也很简单，做一些小修改

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
    .clock-number span {
        display: block;
    }
    .clock-number:nth-child(2) {
        transform: rotate(30deg);
    }
    .clock-number:nth-child(2) span {
        transform: rotate(-30deg);
    }
    ...
</style>
<div id="clock">
    <div class="clock-number"><span>12</span></div>
    <div class="clock-number"><span>1</span></div>
    ...
</div>
{% endhighlight %}

具体代码可以看 [demo 页面](http://crazydogs.github.io/staticpage/css_transform_clock_2.html)，
很简单地就解决了这个问题(当然这些繁复的代码可以用 less 之类的带函数功能的异构语言简化)。

但是有没有更好的方法呢，毕竟这种方案须要修改页面的 DOM 结构，成本比较高，而且元素多了，
对渲染来说也增加了计算量(特别是当大量元素须要这种多重的变换时)。另外，多重嵌套的方案，
如果遇到透明度相关的需求的话，可能会被烦死。

上一期最后讲到了变换合并的内容，那能不能把这两个变换合并，单纯用一个元素做出等价的效果呢。
如果是单纯地合并的话，有一个东西就很麻烦了，那就是变换原点。对于任一个数字来说，须要进行两个方向相反，
角度相同的旋转，这两个旋转操作的原点并不相同，一个是表盘的中心，一个是数字的中心。
而 transform-origin 并不像 tranform 属性一样，支持多个值，一个元素就只有一个
transform-origin 怎么办呢。

本文第一部分中提到说 transform-origin 和 transform: translate 其实是很像的，
那能不能用 translate 来代替 transform-origin 来改变变换的原点的。

还是先考虑最上面提到的正方形缩放的例子。蓝色和红色的正方形之间，差了一些距离，
很容易发现，这段距离其实就是两个变换不同的变换原点之间的距离。那就是说，
如果能找到方法，弥补这段距离，那就可以用平移来实现原点的辩护按了(其实，
仔细想想，就会发现原点的变换就是平移)。

回头看看正方形缩放的代码，其实等价于这样

{% highlight css %}
.div1 {
    transform-origin: 100px 100px;
    transform: scale(0.5, 0.5);
}
.div2 {
    transform-origin: 0 200px;
    transform: scale(0.5, 0.5);
}
{% endhighlight %}

div1 的样式没有指定变换原点，默认值为中心点，在这个例子中就是 **100px 100px**。
div2 的样式指定了变换原点为 left bottom，也就是相当于 **0 200px**。
我们对样式稍加改动变成这样

{% highlight css %}
div {
    transform-origin: 0 0;
}
.div1 {
    transform: scale(0.5, 0.5) translate(100px, 100px);
    background: rgba(0, 0, 255, 0.5);
}
.div2 {
    transform: scale(0.5, 0.5) translate(0, 200px);
    background: rgba(255, 0, 0, 0.5);
}
{% endhighlight %}

就会发现，我们锁定 transform-origin 为元素的左上角的同时，
可以通过一个 translate 的效果来模拟不同原点的变换。Good，
似乎找到方法了，我们把它用到刚刚的表盘例子中

{% highlight css %}
.clock-number:nth-child(2) {
    transform-origin: 0 0;
    transform: rotate(30deg) translate(200px 200px);
}
{% endhighlight %}

就会发现，不好使呀，怎么跑到了一个奇怪的地方 = =，大概是这样

![transform origin demo3](http://crazydogs.github.io/images/css_transform_clock_3.png)

看来靠瞎猜还是不行的，我们须要真正弄明白 origin 和 translate 之间的关系。
还是那句话，欢迎期待下一期。
