---
layout: post
title:  "CSS transform 的基本原理(5)"
date:   2017-04-20 22:13:00
category: coding
---

- [CSS transform 的基本原理(1)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%86.html)
- [CSS transform 的基本原理(2)](http://crazydogs.github.io/coding/2017/04/15/CSS-transform-%E5%8E%9F%E7%90%862.html)
- [CSS transform 的基本原理(3)](http://crazydogs.github.io/coding/2017/04/18/CSS-transform-%E5%8E%9F%E7%90%863.html)
- [CSS transform 的基本原理(4)](http://crazydogs.github.io/coding/2017/04/19/CSS-transform-%E5%8E%9F%E7%90%864.html)

## 从 2D 到 3D

仔细考虑上一篇最后提到的那个问题([demo 地址](http://crazydogs.github.io/staticpage/css_transform_solarsystem.html) 建议用 chrome 打开)，
其实和那个时钟的问题，非常相似，都是一个多层的旋转，用到不同的变换原点。
如果没什么思路，还是要提醒一下，可以参考本系列的第一篇，CSS 的 transform
变换，都是线性变换，是作用于整个空间的。而且由于是线性变换，原点与元素之间的相对位置，
并不会因为平移而改变。

关键的变换代码如下

{% highlight html %}
    <style>
        body, html {
            width: 100%;
            height: 100%;
            background: #000;
        }
        #wrapper {
            position: relative;
            width: 300px;
            height: 300px;
            transform-style: preserve-3d;
        }
        #ball1 {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            background: blue;
            position: absolute;
            top: 130px;
            left: 130px;
            animation: rotateBall1 5s linear infinite;
            perspective: 50px;
        }
        @keyframes rotateBall1 {
            from {
                transform:
                    rotateX(80deg)
                    rotate(0deg) translate(0, -130px)
                    rotate(0deg)
                    rotateX(-80deg);
            }
            to {
                transform:
                    rotateX(80deg)
                    rotate(360deg) translate(0, -130px)
                    rotate(-360deg)
                    rotateX(-80deg);
            }
        }
        #sun {
            width: 50px;
            height: 50px;
            border-radius: 50%;
            background: #ffd679;
            box-shadow: 0 0 40px 4px #ff832b;
            position: absolute;
            top: 125px;
            left: 125px;
        }
    </style>
    <div id="wrapper">
        <div id="ball1"></div>
        <div id="ball2"></div>
        <div id="sun"></div>
    </div>
{% endhighlight %}

我们先来分析一下代码，首先，和表盘一样，我们须要一个以中心为原点的旋转变换，
只不过这次，是动画而不是静态的位置变换。那第一步就是构造一个圆周运动。

{% highlight css %}
#ball1 {
    width: 40px;
    height: 40px;
    position: absolute;
    top: 130px;
    left: 130px;
    animation: rotateBall1 5s linear infinite;
}
@keyframes rotateBall1 {
    from {
        transform:
            rotate(0deg) translate(0, -130px);
    }
    to {
        transform:
            rotate(360deg) translate(0, -130px);
    }
}
{% endhighlight %}

原理很简单，就是将元素旋转一定角度之后进行平移。因为没有指定 transform-origin
所以原点是默认值，也就是元素的中心。选择把元素定位在圆心，而且使用默认原点，
主要是节省接下来操作中不必要的一些平移操作。

接着，和钟表那个问题一样，为了维持元素的方向不变，需要做一个逆向的变换。
这同时也是为了保持元素的 X 轴与初始的 X 轴平行，后面会用到。

{% highlight css %}
@keyframes rotateBall1 {
    from {
        transform:
            rotate(0deg) translate(0, -130px)
            rotate(0deg);
    }
    to {
        transform:
            rotate(360deg) translate(0, -130px)
            rotate(-360deg);
    }
}
{% endhighlight %}

现在这个圆周运动只是在正常的与屏幕平行的平面进行，我们为了突出 3D 的效果，
需要将这个平面倾斜一定角度。

{% highlight css %}
@keyframes rotateBall1 {
    from {
        transform:
            rotateX(80deg)
            rotate(0deg) translate(0, -130px)
            rotate(0deg);
    }
    to {
        transform:
            rotateX(80deg)
            rotate(360deg) translate(0, -130px)
            rotate(-360deg);
    }
}
{% endhighlight %}

因为涉及到了 3D 变换，所以还要指定一个 **transform-style** 属性的值为 **preserve-3d**
这个属性是用于指定子元素变换是否有三维空间的效果。要保证 3D 的效果，浏览器须要做很多额外的工作，
确保正确的元素堆叠顺序，所以如果须要的话，就要明确指定，默认为了提高性能，是不进行这些计算的。

另外还需要指定一个 perspective 属性，这个属性用来指定观察点距离基准平面(z=0)的距离。
其实说到底，这也是一个变换，只不过是一个从三维空间映射到二维空间的变换，
借用一张 [www.cs.sjsu.edu](http://www.cs.sjsu.edu/~bruce/fall_2016_cs_116a_lecture_camera_and_clipping_plane.html)
上面的图

![视锥](http://crazydogs.github.io/images/css_transform_preserve.png)

也就是说，将上图中的形空间中的东西，映射到 near clipping plane。而 preserve
属性就是指定这个平面在 z 轴上的位置。如果不够直观，可以将 demo 中的 preserve
属性改到 200px 以下，感受会明显一些。

其实与视点相关的还有一个属性就是 preserve-oirgin 用来指定视点在 x 轴和 y 轴上的位置，
demo 中的效果也不一定需要用 rotateX 来实现，也可以通过 preserve-origin 拉高视点做到，
具体不占开了。

加了这些东西之后，大概是这样子 [demo2](http://crazydogs.github.io/staticpage/css_transform_solarsystem2.html)

我在这里加了一个平面，方便看清楚究竟发生了什么事情。

现在离我们的目标已经很近了，只要把这个原型立起来就可以了。还记得之前说的，
我们保持了 x 轴与原 x 轴平行吗。而且因为没有指定 transform-origin，
变换原点依然是默认值，圆形的圆心。所以只要在进行一次 X 轴旋转就 OK 了。

{% highlight css %}
@keyframes rotateBall1 {
    from {
        transform:
            rotateX(80deg)
            rotate(0deg) translate(0, -130px)
            rotate(0deg)
            rotateX(-80deg);
    }
    to {
        transform:
            rotateX(80deg)
            rotate(360deg) translate(0, -130px)
            rotate(-360deg)
            rotateX(-80deg);
    }
}
{% endhighlight %}

## 还有什么

基本的东西都已经做完了，那接下来还有什么呢？其实上面给的 demo，还算不上优雅，
太多须要手工编写的 CSS 代码了，而且还没写兼容相关的东西，真正写起来，就是又臭又长了。
所幸现在 CSS 也有很多很好用的工具，可以编写函数，在编译阶段生成代码，也可以借助
JavaScript 来实现自动化，但这可能就是另一个话题了。

另外就是选型的问题，为何使用 CSS 来做这些东西？单元素的实现对比嵌套元素实现的优势，
除了性能之外，还有什么。

使用 WebGL 来绘图，特别是涉及三维空间效果的绘图，性能和表现力当然是远远优于 CSS
的。但它只是一个绘图 API，在交互层面上，是没有默认行为的。如果想在元素上面进行交互，
须要额外地对点击事件进行计算处理，判定落在那个元素上，还是比较复杂。如果使用
tree.js 之类的绘图库，省下了自己处理的麻烦，但却引入了一个不小的 JS 文件，
如果是页面上大面积出现这种绘图需求，使用起来成本才算比较好接受。另一个致命的问题就是，
canvas 元素，不能很好地和其他普通 DOM 元素共存，并不能与其他元素堆叠，透明效果也仅限于
canvas 的内部，这个如果设计上有冲突，是一个没法解决的问题。

CSS 变换，操作的还是 DOM 元素，所以交互层面上，其实跟普通的 DOM 没什么区别，
非常方便。

如果使用多层元素嵌套的话，点击事件有时候会因为元素堆叠，出现诡异的问题，
非常难以调试。遇到与透明度相关的需求时，可能为了透明度不互相影响，有些嵌套变换还不能共享，
导致须要的元素数量更加庞大。

当然使用 SVG 绘图，在交互层面来说，也是非常方便的，只是处理起三维效果，比较吃力，
须要一些 js 的计算。SVG 还是更适合于平面矢量图的一些非线性形变，不过又是另一个话题了。

## Summary

终于写完了这个系列，感觉还不错，整理了一下以前的一些零散的东西。有些东西，
玩一下就算了，可能过段时间，多少会有些忘，虽然没什么人会看，但写出来还是挺爽的。

是时候想想下个主题了。
