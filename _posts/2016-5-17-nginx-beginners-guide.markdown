---
layout: post
title:  "《Nginx Beginner's Guides》"
date:   2016-5-17 14:22:10
categories: jekyll update
---

[原文地址](http://nginx.org/en/docs/beginners_guide.html)

本指南对 nginx 做一个基础的介绍，并提供了一些简单的用例。本文假设 nginx 已经在你的机器上安装了，
如果还没有，请参考[安装指南](http://nginx.org/en/docs/install.html)。本指南内容包括：如何启动/停
止 nginx 服务、应用配置文件更新、配置文件结构的解释、如何配置静态文件转发服务、如何配置代理服务
以及如何连接 fastCGI

nginx 拥有一个主进程和若干个 worker 进程。主进程负责读取、解析配置，管理 worker 进程。worker 进
程负责实际的请求解析。nginx 基于事件机制，依赖操作系统的支持，高效地在 worker 间分配请求处理任务
。worker 进程的数量可以在配置文件中设置或者自动根据 CPU 核数设置。[参考 worker 进程配置](http://nginx.org/en/docs/ngx_core_module.html#worker_processes)

nginx 的工作内容是由配置文件来决定的。配置文件默认为位于 /usr/local/nginx/conf, /etc/nginx 或 /usr/local/etc/nginx
的 nginx.conf。

### 启动、停止 nginx 服务，重新加载配置

通过运行可执行文件来启动 nginx。nginx 启动之后，可以通过 -s 参数对其进行控制，方法如下

    nginx -s signal

signal 可以是以下几种之一
- stop      快速关闭
- quit      退出
- reload    加载新的配置
- reopen    重新打开日志文件

例如等待 worker 进程完成当前请求的处理后停止服务，可以使用

    nginx -s quit
    // 此命令须在开启 nginx 服务的那个用户下执行

对配置文件的修改不会立即被引用。使用 reload 指令或者重启 nginx 才会使其生效，reload 方法如下

    nginx -s reload

主进程收到 reload 指令之后，会先对新的配置文件进行语法检查，然后尝试引用其中的配置。如果成功了，
主进程会启动新的 worker 进程，并向旧的进程发出停止指令，否则主进程回滚到旧的配置。旧的 worker
收到停止命令之后，就停止接受新的请求，完成当前的请求服务后，进程就停止了。

nginx 还会收到来自 kill 之类的 Unix 工具的指令。这种情况下，指令直接发送给指定 ID 的进程。nigx
主进程的 id 默认在位于 /usr/local/nginx/logs 或者 /var/run 下的 nginx.pid 中指定。例如，主进程
的进程 id 是 1628，那可以通过以下命令发送 quit 指令

    kill -s QUIT 1628

可以通过 ps 工具来获取正在运行中的 nginx 进程，如下

    ps -ax | grep nginx

更多指令相关的信息，可以[参考控制 nignx](http://nginx.org/en/docs/control.html)

### 配置文件结构

nginx 模块组成是由配置文件中的指令控制的。指令分为单行指令和指令块。一个单行指令由指令名和参数组
成，中间用空格隔开，行末用分号结束。指令块的结构也是类似的，但与单行指令不同，行末不是用分号结束
，而是附加一些由花括号({})括起来的指令。如果一个块指令能够将其他指令包括在块内，那么称它是一个上
下文（例如 evnets, http, server, location）.

在配置文件中，不在任何上下文内的指令，视为处于 main 上下文中。evnts 和 http 指令处于 main 上下文
中，server 在 http 中，location 在 server 中。

在井号后面直到行末的内容是注释。

#### 处理静态内容

服务器一个很重要的任务就是处理文件请求(例如图片或者静态的 html 页面)。接下来会实现一个示例，根据
请求不同，nginx 会转到不同的本地目录: /data/www(存储 HTML 文件) 和 /data/images(存储图片)。实现这
个须要修改 nginx 的配置文件，在 http 块下创建一个 server 块，server 块又包含两个 location 块

首先，创建 /data/www 目录并在其中新建 index.html 文件，随便写上些内容，创建 /data/images 目录并在
其中放几张图片。

然后打开配置文件。默认的配置文件里面已经有几个示例的 server 指令块，大部分都是注释状态。将所有的
server 快注释掉，然后新建一个 server 块

    http {
        server {
        }
    }

通常来说，配置文件会包括几个 server 块并通过监听端口的不同，还有 server 的名称来区分。nginx 确定由
哪个 server 来处理请求后，就把在请求头里的 URI 拿出来，匹配 server 块中 location 指令的参数。

添加下面这个 location 块到 server 块中

    location / {
        root /data/www;
    }

这个 location 块指定了 "/" 前缀来匹配请求的 URI。匹配这个前缀的 URI 会加到 root 指令指定的路径，也
就是 /data/www 后面，来组成请求指定的文件在本地的路径。如果同时有多个 location 块能匹配到请求的 URI
那 nginx 会选取前缀最长的那一个。上面这个 location 块提供了最短的前缀，所以只有当其他 location 块都
不匹配的时候，才会使用这个块。

接下来添加第二个 location 块

    location /images/ {
        root /data;
    }

这个块会匹配以 /images/ 开头的请求(location / 也能匹配这样的请求，但是前缀更短)

server 块配置后应该是这样

    server {
        location / {
            root /data/www;
        }
        location /images/ {
            root /data;
        }
    }

这已经是一个能够工作的服务配置了，默认监听 80 端口。在本机上可以通过 http://localhost/ 访问。如果请求
URI 是以 /images/ 开始的话，服务会返回 /data/images 下的文件。比如说，收到请求 http://localhost/images/example.png
nginx 会返回 /data/images/example.png 文件。如果文件不存来，nginx 会返回 404 错误。不是以 /images/开
头的请求则会被映射到 /data/www 目录。例如，请求 http://localhost/some/example.html，nginx 会返回 /data/www/some/example.html
文件。

要让新的配置生效，须要启动 nginx 或者发送 reload 指令到 nginx 的主进程，如下

    nginx -s reload
    如果发生什么问题，可以从位于 /usr/local/nginx/logs 或 /var/log/nginx 下的 access.log 和 error.log
    中寻找原因。

### 创建一个简单的代理服务

nginx 一个常见的需求是把它设置成一个代理服务。代理服务的意思是，服务器接到请求，把它传给一个代理服务器
，接收代理服务器的返回，传递给客户端。接下来我们会配置一个基础的代理服务，将图片请求指向本地目录，其他
请求发送到代理服务器。在这个例子中，所有服务都在一个 nginx 实例中定义。

首先，在配置文件中增加以下配置

    server {
        listen 8080;
        root /data/upl;

        location / {
        }
    }

这是一个监听 8080 端口的服务(在之前的教程中，没有使用 listen 指令，都是使用默认 80 端口)，把所有请求都
转到 /data/upl 这个本地目录。创建这个目录并在其中创建一个 index.html 文件。root 指令写在 server 块中，
只有当 location 块中没有 root 指令的时候，才会使用这个 root。

接下来，修改这个基本配置，让它成为一个真正的代理服务配置。在第一个 location 块中，使用 proxy_pass 指令
参数是带有 协议名、域名、端口的地址（在本例中，是 http://localhost:8080）

    server {
        location / {
            proxy_pass http://localhost: 8080;
        }
        location /images/ {
            root /data;
        }
    }

然后开始编辑第二个 location 块。现在这个 location 是匹配所有带 /images 前缀的请求到 /data/images 目录，为了
让它能够匹配带有特定文件后缀的图片请求，做如下修改

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }

指令的参数是一个用来匹配以 .gif, .jpg 或 .png 结束的请求。正则表达式前需要 ~ 进行标识。

当 nginx 决定使用哪一个 location 块来处理一个请求时，它会先检查使用前缀的 location，再检查使用正则的，当然
之前说过使用前缀的 location 中，优先使用前缀长的。

最后我们的代理配置会是这样

    server {
        location / {
            proxy_pass http://localhost:8080/;
        }
        location ~ \.(gif|jpg|png)$ {
            root /data/images;
        }
    }

这个服务会拦截以 .gif, .jpg, .png 结尾的请求，将他们匹配到 /data/images 目录(通过将 URL 加到 root 指令的参数
后面)，然后把剩下的其他请求发送到代理服务器。

要让新的配置生效，须要发送 reload 指令给 nginx，具体方法上面已经讲过了。

当然，还有许多[其他命令](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)可以用来配置代理服务

### 配置 FastCGI 代理


