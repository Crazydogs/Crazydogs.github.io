---
layout: post
title:  "Node 中的流 - Writable 与其他"
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
            // chunk 数据块
            // encoding 编码格式
            // next 回调函数
            someDataBase.save(chunk).then(() => {
                next();
            });
        }
    });
{% endhighlight %}

当处理完当前的数据块后，使用回调函数，通知此数据块已经处理完毕。

writev 函数是可选的，当你想要有一次处理多个数据块的能力的时候，可以实现这个方法，
stream 会使用当前缓存区内所有的数据块作为参数来调用它。writev 接受两个参数，
*writev(chuncks, callback)*，其中 callback 是回调函数，和 write 一样，数据处理完之后，
调用此函数。chunks 是一个数组，其中每一个元素的格式像这样 *{chunk: ..., encoding: ...}*
其实就是 write 函数的批量处理版本。

有了 Readable 和 Writeable 之后，其他两种就很好理解了。Duplex 是同时实现了 Readable
和 Writable 的 stream，比如 TCP socket(输入与输出之间并不相关，仅仅是双工)，
而 Transform 是 Duplex 中的一种，输入和输出之间是有依赖关系的，相当于一个 filter，
将数据进行变换之后传给后面的环节。

比如说我们想要使用流读取文件内容，但又不想使用 size 这种粒度，想要一个数据块就是一行
（当然如果你知道 readline 模块的话，就先假装不知道吧）应该怎么做呢。很自然就会想到使用一个
Transform 流整理数据的格式。

{% highlight javascript %}
    let fs = require('fs');
    let stream = require('stream');
    let util = require('util');
    let process = require('process');

    let fileStream = fs.createReadStream(
        '/path/to/file',
        {encoding: 'utf8'}
    );

    util.inherits(Chunk2Line, stream.Transform);
    function Chunk2Line(opt) {
        stream.Transform.call(this, opt);
        this._strBuffer = '';
        this.count = 1;
    }
    Chunk2Line.prototype._transform = function(chunk, encoding, callback) {
        let lines = chunk.split('\n');
        if (lines.lenght > 1) {
            this._strBuffer = lines.pop();
        }
        lines.map(item => {
            this.push(this.count + ' ' + item + '\n', 'utf8');
            this.count++;
        });
        callback();
    };
    let chunk2Line = new Chunk2Line({decodeStrings: false});

    fileStream.pipe(chunk2Line).pipe(process.stdout);
{% endhighlight %}

代码大概像上面这样，我们就拥有了以行为单位的流，并给每一行加上了行号。
Transform 流的变换功能非常灵活，可以很方便的组装成想要的管道，
而且一些 Transform 的功能很容易抽取出纯函数来，就可以实现整个数据流程的分阶段缓存，
应用场景其实是很丰富的。

Transform 流还有一个重要的 flush 函数，和其他的一些 API，就不多赘述了，还是看文档靠谱。
- [官方文档](https://nodejs.org/dist/latest-v6.x/docs/api/stream.html)
- [一个还不错的翻译](https://github.com/Crazydogs/node-doc/blob/v6/doc/stream.md)
