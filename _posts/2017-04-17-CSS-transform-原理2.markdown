---
layout: post
title:  "CSS transform 的基本原理(2)"
date:   2017-04-15 22:13:00
category: coding
---

[CSS transform 的基本原理(1)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%86.html)

## 小试牛刀

上一篇讲了一些变换矩阵相关的东西，反正我是觉得挺有趣的。但，有什么用呢？

来介绍一个简单的应用吧，如果你须要用 CSS 的 transform 完成一个水平翻转，
你会怎么做呢？2D 变换中只提供了缩放、斜切、旋转、平移四个操作，并没有水平翻转。
这时候就可以使用 matrix 属性。

按上一篇所说的方法，我们考虑水平翻转会对单位向量产生什么影响。其实就是 x
轴的单位向量，方向取了个反。所以水平翻转的变换矩阵是

![公式1](http://crazydogs.github.io/images/css_transform_2_1.png)

相应的 CSS 代码就是

{% highlight css %}
div {
    transform: matrix(-1, 0, 0, 1, 0, 0);
}
{% endhighlight %}

当然，机智的朋友可能会说，直接用 3D 变换中的旋转操作，180° 刚好就是水平翻转嘛，
也没有必要用到 matrix 属性啊。确实没错，这也是水平翻转操作名称的由来，而且，
这种用高维空间解决低维空间的思想，也非常酷。

但这样，又带来了一个新的问题。如果有个需求，是要把一个 3 维空间从左手系翻转成右手系呢
(好啦，我知道根本没什么可能碰到这种需求，只是举个例子)，考虑 transform 提供的
3D 变换操作，同样是这 4 个: 缩放、斜切、旋转、平移。这些操作，并没有翻转空间的效果，
而且 CSS 也没有提供 4 维变换的接口(不过即使提供了 4 维的接口，弄明白如何使用能够对应
3 维的翻转，要花的时间估计不比这些线性代数少)，不能像之前那样用降维打击的方式完成了。

不过相信看到这里，应该都能很简单的使用 matrix3d 属性解决这个问题了。

## 迈向 3D 变换

matrix3d 和 matrix 其实非常像，只不过这个矩阵，是一个 4 * 4 矩阵，而且 16
个位置都开放给开发者设置(matrix 只有 6 个参数，而不是 9 个)。参数是按列填入矩阵的。
也就是说

{% highlight css %}
div {
    transform: matrix(a1, b1, c1, d1, a2, b2, c2, d2, a3, b3, c3, d3, a4, b4, c4, d4);
}
{% endhighlight %}

对应的矩阵为

![公式2](http://crazydogs.github.io/images/css_transform_2_2.png)

同样的，a4、b4、c4 三个参数用来做平移操作。你看，用矩阵还有一个好处，
就是升高维度并不需要什么额外的工作去理解 transform 的机制，都只是变换矩阵而已。

## 走进 transform-origin

但事实上，transform 中还有一个东西会影响到变换的结果，那就是 transform-origin，
变换的原点。
