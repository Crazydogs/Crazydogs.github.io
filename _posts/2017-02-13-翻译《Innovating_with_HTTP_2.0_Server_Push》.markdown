---
layout: post
title:  "翻译《Innovating with HTTP 2.0 Server Push》"
date:   2017-02-13 16:44:10
category: coding
---

[原文地址](https://www.igvita.com/2013/06/12/innovating-with-http-2.0-server-push/)

**HTTP2 允许服务器对单个客户端请求并行返回多个响应，这也被称为服务端推送。**
为什么我们须要这样的功能呢？因为通常来说一个页面需要几十个额外的资源，如 JavaScript，
CSS 和各种图片，所有这些资源的引用都内嵌在服务端生产的HTML中。
如果服务端能直接向客户端发送这些资源，而不是等待客户端解析 HTML 再发现须要那些资源，
并进行请求，就可以减少一些不必要的网络请求延时。

实际上，**如果你试过在 HTML 中内联资源(CSS, js, 图片等)，那你已经模拟过服务端推送了**
(一个内联资源被作为文档的一部分进行推送)。唯一的区别是 HTTP2.0 让这个机制更加强大。

### 手动实现 HTTP 2.0 服务端推送
根据定义，一个内联文档是父文档的一部分，因此，它不能单独地被缓存，
但又可能需要在许多不同的页面上复制，这样就非常地低效。而通过服务端推送的内联资源，
是可以实现跨多个页面的缓存的，举个例子：

````
spdy.createServer(options, function(req, res) {
  // push JavaScript asset (/main.js) to the client
  res.push(
    '/main.js',
    {'content-type': 'application/javascript'},
    function(err, stream) {
        stream.end('alert("hello from push stream!")');
    }
  );

  // write main response body and terminate stream
  res.end('Hello World! <script src="/main.js"></script>');
}).listen(443);
````

上面是一个使用 node-spdy 模块实现的最小 SPDY 服务器，对所有的请求都回返回一个
"Hello World!" 字符串外加一个 script 标签。除此之外我们还做了些别的：将
main.js 文件推送到客户端，其内容是触发一个 alert 弹窗。

这样，当浏览器在 HTML 中发现 script 标签时，main.js 文件就已经在缓存中了，
不会产生额外的网络往返。可见 HTTP 2.0 的服务端推送完爆内联。最重要的是，所有支持
SPDY 的浏览器(Firefox，Opera和Chrome) 都支持服务端推送。

### 还能推送什么？
使用服务端推送代替内联资源是标准做法，但为什么止步于此呢，我们还能推送些什么？
可以利用它推送重定向吗？当然，这也很简单。

````
spdy.createServer(options, function(req, res) {
  //push JavaScript asset (/newasset.js) to the client
  res.push(
    '/newasset.js',
    {'content-type': 'application/javascript'},
    function(err, stream) {
      stream.end('alert("hello from (redirected) push stream!")');
    }
  );

  // push 301 redirect: /asset.js -> /newasset.js
  res.push(
    '/asset.js',
    {':status': 301, 'Location': '/newasset.js'},
    function(err, stream) {
      stream.end('301 Redirect');
    }
  );

  // write main response body and terminate stream
  res.end('<script src="/asset.js"></script>');
}).listen(443);
````

和之前的例子差不多，只是我们将 asset.js 资源 301 重定向到 newasset.js。
浏览器负责缓存和重定向资源，代码将会被执行而且没有额外的网络请求产生。
应用程序？这个我们留给读者作为练习内容。

服务端推送还可以更进一步吗？推送缓存失效或者更新缓存到客户端？
我们可以通过将在客户端已存在的缓存标记为过期(推送旧的时间戳)，或者相反地，
推送具有未来过期时间的 304 来更新缓存。总之，服务端可以主动地管理客户端的缓存。

如果服务端的行为过于激进或者出现错误，客户端可以限制推送流的数量，甚至取消指定的流。
找到正确的策略还有双方的平衡点是取得服务端推送最佳性能的关键点，这不容易做到，
但可以产生很高的回报。

注意：当前的浏览器并不支持推送缓存更新，这合理吗？

### 使用服务端推送进行客户端通知
服务端推送不能代替 Server-Sent Events (SSE) 或 WebSocket 这些技术。通过服务端推送的资源，
由浏览器进行处理，而不会冒泡到应用的代码，没有 JavaScript 的 API 来获取这些事件的通知。
解决这个问题很简单，我们可以结合 SSE 通道和服务端推送，得到更好的效果。

````
spdy.createServer(options, function(req, res) {
  // set content type for SSE stream
  res.setHeader('Content-Type', 'text/event-stream');

  messageId = 1;
  setInterval(function(){
    // push a simple JSON message into client's cache
    var msg = JSON.stringify({'msg': messageId});
    var resourcePath = '/resource/'+messageId;
    res.push(resourcePath, {}, function(err, stream) { stream.end(msg) });

    // notify client that resource is available in cache
    res.write('data:'+resourcePath+'\n\n');
    messageId+=1;
  }, 2000);
}).listen(443);
````

诚然，这个例子很蠢，但主要是为了说明。服务端周期性地以两秒为间隔，产生一个消息，
推送到客户端内存中，然后发送 SSE 通知客户端，然后客户端就可以在应用代码中订阅这些事件，
对其进行处理

````
<script>
  var source = new EventSource('/');

  source.onmessage = function(e) {
    document.body.innerHTML += "SSE notification: " + e.data + '<br />';

    // fetch resource via XHR... from cache!
    var xhr = new XMLHttpRequest();
    xhr.open('GET', e.data);
    xhr.onload = function() {
      document.body.innerHTML += "Message: " + this.response + '<br />';
    };

    xhr.send();
  };
</script>
````

SSE 长连接，服务端推送，和其他的 HTTP2.0 流，高效地复用同一个 TCP 连接，没有额外的连接、
往返开销。通过 SSE 和服务端推送相结合，我们可以向客户端提供任意资源(二进制内容或文本)，
利用浏览器缓存并执行对应的处理程序。最棒的是，这些可以在当前使用。

### 服务端推送的革新
服务端推送提供了很多新的优化的可能性。上面的例子只是说到一些很浅的东西，
还有很多其他问题须要考虑。

- 哪些资源应该被推送，何时被推送？资源是否已经被缓存
- 服务端能否自动推断出哪些资源须要推送
- 更复杂的应用，如主动的缓存管理，是否应该被支持？
- 我们如何设计我们的应用程序以充分利用服务器推送?
- ...

有了 HTTP2.0 服务端可以变得远比以前智能，优化对单个资源的交付，甚至是分发整个应用。
浏览器可以暴露更多的 API 和能力，让这个过程更加高效。事实上，从长远来看，
服务端推送可能是 HTTP2.0 的杀手特性！

PS: 想了解更多关于HTTP 2.0？ 查看我的O'Reilly书的
[免费预览](http://chimera.labs.oreilly.com/books/1230000000545?utm_source=igvita&utm_medium=referral&utm_campaign=igvita-body)
