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


