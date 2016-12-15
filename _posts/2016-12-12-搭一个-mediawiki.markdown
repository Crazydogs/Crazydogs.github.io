---
layout: post
title:  "搭一个 MediaWiki"
date:   2016-12-12 17:48:00
category: coding
---

在公司写 PHP 一直都是用 ODP，自己的 PHP 水平基本上就是能做语言翻译那样，还要经常查手册的那种。
总觉得 PHP 不是很适合我，也不知道这种主观感觉到底哪里来的 = =。

刚好碰到一个需求，须要搭一个内部的文档平台。自己写的话工期实在太长了，也没必要，所以决定试一试
MediaWiki。现在这种东西，应该都是傻瓜化的了吧，给自己估了一天的工作量（= = 还是 too
young）。

## PHP 版本
MediWiki 须要的 PHP 版本是 5.5.9。而手头上机器的 ODP 带的是 2 点几的，又不敢随便升级，
实在是太老了，人家都到 7 了啊。当然自己的环境应该不会遇到这种问题。
所以只能自己另外搭一个环境了，手动搭的话还是太累了，选择了用 XAMPP 来做。

## XAMPP
XAMPP 这种东西真是造福我这种新手玩家，传说中的一键安装，就是上[官网](https://www.apachefriends.org/zh_cn/index.html)
下个包，然后 run 就行了。

但其实还是碰到了不少问题。
1. root 权限
    XAMPP 安装是须要 root 权限的，公司很多机子并不会给 root。
2. 64 位系统
    XAMPP 不支持 64 位的系统，须要安装一个 ia32-libs 库来进行兼容。
3. 系统版本
    机器上的系统很老了，是 CenterOS 3 的，安装之后总是会报 GCC 和 GLIBC 加载的错误。
    只能整个重装。

幸好最后找到了一个有 root 的 ConterOS5 机器，不然真是不知道折腾到啥时候

## MediaWiki
基本上就是照着[安装指南](https://www.mediawiki.org/wiki/Manual:Installation_guide)
做就可以完成了。不过也不是一帆风顺

1. 数据库
    如果有数据库的 root 权限可以不用管，直接下一步就好了。如果没有 root 权限的话，
    须要事先创建一个库，并在安装过程中指定。
2. 权限管理
    其实之前没怎么用过 MediaWiki 这一套，并不是很熟悉，了解之后其实还是很简单的。
    所有的配置都包含在项目根目录下的 LocalSettings.php 文件里。权限管理其实就是配置
    $wgGroupPermissions 这个多维数组。在『特殊:用户组权限』这个页面下可以看到当前的权限状态，
    也可以看到每个权限的名字，根据自己的须要配置就好了。
    类似这样，普通用户是 user，管理员 sysop，行政人员 bureaucrat
    ````
        $wgGroupPermissions['*']['createaccount'] = false;
        $wgGroupPermissions['*']['edit'] = false;
        $wgGroupPermissions['*']['read'] = false;
        $wgGroupPermissions['*']['writeapi'] = false;
        
        # 普通用户只能进行 read 操作
        $wgGroupPermissions['user']['edit'] = false;
        $wgGroupPermissions['user']['createpage'] = false;
        $wgGroupPermissions['user']['movefile'] = false;
        $wgGroupPermissions['user']['move'] = false;
        $wgGroupPermissions['user']['move-categorypages'] = false;
        $wgGroupPermissions['user']['move-rootuserpages'] = false;
        $wgGroupPermissions['user']['move-subpages'] = false;
        $wgGroupPermissions['user']['upload'] = false;
        $wgGroupPermissions['user']['reupload'] = false;
        $wgGroupPermissions['user']['reupload-shared'] = false;
        $wgGroupPermissions['user']['writeapi'] = false;
        $wgGroupPermissions['user']['changetags'] = false;
        $wgGroupPermissions['user']['purge'] = false;
    ````
3. 邮件服务
    公司内网邮箱的安全策略问题，须要使用特定的 SMTP 服务，须要在配置文件中设置
    $wgSMTP 这个变量。

    ````
            $wgSMTP = array(
                'host' => "邮箱域名",
                'IDHost' => "发送域名",
                'port' => 25,
                'auth' => true,
                'username' => '发件用户名',
                'password' => '密码',
            );
    ````

## EOF
估计工作量的时候，还是太年轻了，特别是涉及到环境问题这种玄学领域的时候，
真的是说不清楚。
