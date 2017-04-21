---
layout: post
title:  "CSS transform 的基本原理(4)"
date:   2017-04-19 22:13:00
category: coding
---

- [CSS transform 的基本原理(1)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%86.html)
- [CSS transform 的基本原理(2)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%862.html)
- [CSS transform 的基本原理(3)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%863.html)

## transform-origin 与 translate

还是用上一篇提到的表盘的例子([demo地址](http://crazydogs.github.io/staticpage/css_transform_clock_1.html))。初始状态是下图这样。

![transform origin demo4](http://crazydogs.github.io/images/css_transform_clock_4.png)

这里面的坐标系跟平时用的会有点不同，因为 CSS 中的坐标，是从左上开始的，
向下和向右为正方向。另外就是我们为了让数字居中，所以其实数字元素是占整行的，
也就是图中的灰色矩形。

我们须要把这个数字 1 以表盘圆心(也就是坐标为 200, 200 的点)为中心，旋转 30°。
使用 transform-origin 的话就非常简单，只需要这样

{% highlight css %}
{
    transform-origin: 200px 200px;
    transform: rotate(30deg);
}
{% endhighlight %}

那要如何才能不使用 transform-origin 实现这个效果呢

![transform origin demo5](http://crazydogs.github.io/images/css_transform_clock_5.png)

看一下上面这张图的几条辅助线，是不是已经想到了！再提示一点吧，如果你还没有弄清楚线性变换是怎么回事，
首先最重要的事认真思考一点。变换是针对目标元素(这个例子中也就是灰色长方形)的变换吗？
如果还不确定，回去看看本系列的第一篇，变换是对整个空间的变换。也就是说，
旋转变换不单单让矩形倾斜了，也让矩形所在的空间倾斜了。而且由于是使用高维空间的
skew 斜切变换来进行平移操作，所以平移也并不会改变元素与原点之间的相对位置。

考虑这样的变换

{% highlight css %}
{
    transform-origin: 0 0;
    transform: translate(200px, 200px) rotate(30deg);
}
{% endhighlight %}

会得到下图右边的结果

![transform origin demo6](http://crazydogs.github.io/images/css_transform_clock_6.png)

还记得我们刚才说的，旋转变换让矩形所在的空间也倾斜了吗，接下来只要再加上一个平移变换

{% highlight css %}
{
    transform-origin: 0 0;
    transform: translate(200px, 200px) rotate(30deg) translate(-200px, -200px);
}
{% endhighlight %}

噔噔！完成了！

![transform origin demo7](http://crazydogs.github.io/images/css_transform_clock_7.png)

如上图所示，我们使用 translate 完美地代替了指定原点的变换。这里只演示了旋转变换，
剩下的就留给大家想象了，transform-origin 究竟代表了什么意义。

## 更进一步

有了上面的理论基础，我们就可以对之前的钟表 demo 进行改进了，消除掉多余的一层元素，
实现想要的双重旋转的效果。代码类似这样

{% highlight css %}
    .clock-number:nth-child(2) {
        transform:
            translate(200px, 200px)
            rotate(30deg)
            translate(-200px, -200px)
            translate(200px, 20px)
            rotate(-30deg)
            translate(-200px, -20px);
        transform-origin: 0 0;
    }
{% endhighlight %}

具体代码可以看[这里](http://crazydogs.github.io/staticpage/css_transform_clock_3.html)，
实际开发中可以使用 less 之类的工具缩减一下重复的代码。

可以注意到，我们连着使用了两次上面提到的技巧，完成了两次原点不同的旋转操作。
停下来认真思考一下，是什么确保了这个技巧可以重复使用的。

接下来，我们还可以更进一步，想象一下，像这样一个 [demo](http://crazydogs.github.io/staticpage/css_transform_solarsystem.html)
应该如何实现，才算是比较优雅呢。

## Summary

下一期就是最后一期了，第一次写这么长篇的文章。果然要把一件事情讲清楚并不简单，
写出来还是很考验功力的。要有扎实的基础和系统的认识，才能写得比较好吧，
感觉自己还差些火候。

还是期待下一期吧。

- [CSS transform 的基本原理(5)](http://crazydogs.github.io/coding/2017/04/19/CSS-transform-%E5%8E%9F%E7%90%865.html)
