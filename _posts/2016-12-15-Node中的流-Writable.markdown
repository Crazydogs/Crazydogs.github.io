---
layout: post
title:  "Node 中的流 - Writable"
date:   2016-12-15 21:25:10
category: coding
---

 [Readable 流](http://crazydogs.github.io/coding/2016/12/04/Node%E4%B8%AD%E7%9A%84%E6%B5%81-Readable.html) 

 除了 Readable 流之外，当然还有 Writable 流。

{% highlight javascript %}
    let stream = require('stream');

    let myWritable = new stream.Writable({
        // 内部缓存大小，默认为 16384(16kb)，对于处于 objectMode 的流，默认是 16(个对象)
        highwatermark: 16384,
        // 布尔值，决定是否在传给 _write 之前将数据转为 buffer，默认为 true
        decodeStrings: false,
        // 是否为对象模式
        objectmode: false,
        write: _write 函数的实现,
        writev: _writev 函数的实现
    });
{% endhighlight %}

highwatermark 参数和 Readable 流是一样的，标识内部缓存区的大小。如果被打满了，
使用 write 方法写入数据的话，是会返回 false 的。而当缓冲区的数据被消耗，
重新获得可写入的空间时，Writable 流就会发出 drain 事件，通知说可以写入了，跟
Readable 流的 readable 事件正好相反。

第二个参数 decodeStrings 如果不设置的话，在 \_write 函数中收到的数据块都会是 Buffer 的形式，
如果想要拿到字符串进行处理，那就须要将其设置为 false。

淡然最重要的还是 write 参数，比如说想把收到的数据写入数据库。大概会是这样

{% highlight javascript %}
    let stream = require('stream');

    let myWritable = new stream.Writable({
        objectMode: true,
        write(chunk, encoding, next) {
            someDataBase.save(chunk).then(() => {
                next();
            });
        }
    });
{% endhighlight %}

当处理完当前的数据块后，使用回调函数
