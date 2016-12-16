---
layout: post
title:  "Node 中的流 - Readable"
date:   2016-12-04 21:25:10
category: coding
---

最近又做了一些日志回收，入库之类的工作。用了之前一直没什么机会用到的 stream 模块。
stream 模块 API 变化很大啊，比之前好多了，也稳定下来了，之前居然连 pause
方法都没有，就像开了水龙头就关不上一样，只能自己拿杯子接着。

Node 中的流分 Readable，Writable，Duplex，Transform 几种。平时其实也多多少少会用到，
因为 Http 请求和返回都是 stream，只是说常用 express，包了一层，并不是很关注。

除了 Http 之外，stream 用的最多还是文件读写了，现在 fs 模块也相应地提供了 fs.createReadStream
和 fs.createWriteStream 两个方法，非常地方便。

先来看一下怎么创建一个流

{% highlight javascript %}
    let stream = require('stream');

    let myReadable = new stream.Readable({
        // 内部缓存大小，默认为 16384(16kb)，对于处于 objectMode 的流，默认是 16(个对象)
        highwatermark: 16384,
        // 默认为 null，如果设置了编码，buffer 会被转换为指定格式的字符串
        encoding: 'utf8',
        // 是否为对象模式
        objectmode: false,
        read: _read 函数的实现
    });
{% endhighlight %}

Node 中的流对象，大概可以理解为一个水管中的阀门一样的东西，流对象内部会有一个缓冲区，用来暂存数据，
上面的参数中的 highwatermark 就是用来设置缓冲区的大小的。像阀门一样，内部的容积，
相对于整体数据是很小的，只是起到一个暂存缓冲的作用。

Readable 的作用一般是从数据源中汲取数据，供给整个数据流后面的部分使用，相当于对数据进行了一次转换，
由其他的形式（比如文件，数据库等）转换为流的形式。当然它也可以自己生成数据。

流有着两种状态，一种是暂停状态，一种是流状态，这也是我觉得现在 stream 模块可用性提高了很多的主要原因。
暂停状态就像我们家里水龙头接水一样，平时是不出水的，须要的时候自己拿容器去接一点
（调用 Readable 的 read 方法），而流模式就像拿水管在花园里浇花一样，水龙头是一直开着的，
水有多少出多少，把关掉水流的操作留给了后面的环节（也就是说后面如果堵住了，数据的流动也会停止）。
流默认是暂停模式的。

转换流对象的状态的方法有很多，最直接的是两个方法

{% highlight javascript %}
    stream.pasue();     // 暂停
    stream.resume();    // 恢复流模式
{% endhighlight %}

但还有其他一些方法会导致模式的切换

{% highlight javascript %}
    // 添加对 data 事件的监听就是向流索要数据，会自动切换到流模式
    // 尽可能快地获取或生成数据
    myReadable.on('data', chunk => {
        // do somthing
    });
    // 通过 pipe 发送数据也是一样的，会自动切换到流模式
    myReadable.pipe(someWritableStream);
{% endhighlight %}

如果须要切换回暂停模式的时候，须要移除对 data 事件的监听，还要调用 pasue() 方法。

从 Readable 流中获取数据，实际上是调用了对象内部的 \_read 方法，也就是上面代码里面的配置项中的 read。
read 接收一个参数 size，表示要读取数据的数量，然后须要使用 push 方法向缓存区中推入数据。

比如我们创建一个只会返回字符串 '123' 的可读流，那么他的 \_read 方法可以是这样

{% highlight javascript %}
    let stream = require('stream');
    let process = require('process');

    let myReadable = new stream.Readable({
        // 内部缓存大小，默认为 16384(16kb)，对于处于 objectMode 的流，默认是 16(个对象)
        highwatermark: 16384,
        // 默认为 null，如果设置了编码，buffer 会被转换为指定格式的字符串
        encoding: 'utf8',
        // 是否为对象模式
        objectmode: false,
        read(size) {
            this.push('123', 'utf8');
        }
    });
    myReadable.pipe(process.stdout);
{% endhighlight %}

如果运行这个文件，就会在你的标准输出狂刷 123 了。但如果须要复杂一些的功能，
可能就不能这样直接用构造函数生成了，可以使用 util 模块的 inherits 函数来继承 Readable
创建自定义的流，像这样

{% highlight javascript %}
    let Readable = require('stream').Readable;
    let util = require('util');
    let process = require('process');

    util.inherits(customRead, Readable);
    function customRead(opt) {
        Readable.call(this, opt);
        this.max = 10;
        this.cunrrent = 1;
    }
    customRead.prototype._read = function (size) {
        if (this.cunrrent <= this.max) {
            this.push(String(this.cunrrent++), 'utf8');
        } else {
            // null 代表后面已经没有其他数据了，可读流会结束，触发 'end' 事件
            this.push(null);
        }
    }

    let test = new customRead();
    test.on('end', () => {
        console.log('end');
    });
    test.pipe(process.stdout);
{% endhighlight %}

执行这个文件的话，就会输出 1 到 10 然后数据 end。

## EOF
当然 Readable 流还有很多方法和事件，比如 close 事件，readable 事件，setEncoding
方法，unpipe 方法之类的具体就看 node 的[文档](https://nodejs.org/docs/latest/api/stream.html)
吧。（最近发现还有个[中文文档](https://pinggod.gitbooks.io/nodejs-doc-in-chinese/content/doc/stream.html)
翻译得还挺不错的）
