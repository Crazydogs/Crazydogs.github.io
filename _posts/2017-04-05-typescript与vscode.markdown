---
layout: post
title:  "TypeScript 与 VSCode"
date:   2017-04-05 16:44:10
category: coding
---

## TypeScript

TypeScript 虽然很久以前就一直有听说，但是都没有好好去了解。毕竟 JavaScript 写习惯了，
强类型的东西用起来总感觉束手束脚的。不过回头想想，都用了这么久的 const 了，
其实跟强类型也没啥差别了，实际上比 TypeScript 还要严格。算上接口定义和类型检查，
反而 TS 更有优势些。

另一个契机是最近更新了 VSCode，感觉现在插件数量上来了，用起来爽了很多，
终于可以基本代替 vim 的功能了，配合着使用 TypeScript 体验更加舒爽。

这几年前端在折腾的，很多都是工程化的问题。由于 JS 语言的先天性缺陷，前端项目规模大了之后，
就会变得很难管理。虽然有了很多民间方案通过编译来部分实现 ES 的新标准，但很多涉及语言机制的东西，
还是很吃力。而且，似乎比起 Java，C++ 之类的工程化程度，还是差的很远。

TypeScript 的意义更多在于编译之前的开发过程，或者预编译阶段，配合支持得比较好的
IDE 的话，就能做到即时语法检查。

到现在为止，其实工作中碰到的前端项目，并没有那么大型。前端加起来也就五六个人，
通过 code review，和一些文档工作，沟通上还没有遇到很多问题。但人更多一些，
就开始出问题了，总会有不喜欢写文档的人，也总会有不喜欢看文档的人。JS
太过自由可能就变成一种负担。

TypeScript 正如名字所说，最重要的就是给 JS 加上了类型概念。而且加的方法很巧妙，
单从类型上来说，实际上很多情况 ts 文件转 js 文件代码都并没什么变化，定义的 Interface
甚至都不会出现在产出中，而只是在编译之前对代码进行语法分析，在分析的过程中，
加上了类型的限制。这种方式保证了 TypeScript 非常完美的向后兼容，一种严格意义上的
JavaScript 超集，从 JS 迁移过来的成本非常之低。

增加了类型检查和 Interface 的机制之后，其实就能部分解决文档之类的问题了
（当然又变成了不愿意写注释的问题了，不过至少好一些），配合上编辑器，可以达到下图这样的效果。

![补全提示](http://crazydogs.github.io/images/typescript1.png)

## 配合 VSCode

当然仅有自己的代码有这种效果还是不够的，所幸现在 node 的模块和很多主流的库，
对 tsd 的支持程度都很不错了。在项目目录中安装 @types/... 的包，就可以很简单地得到库的描述文件。

比如在项目目录中执行 **npm install --save @types/node**，安装描述文件包，
在 TypeScript 配置文件 [tsconfig.json](https://zhongsp.gitbooks.io/typescript-handbook/content/doc/handbook/tsconfig.json.html#types-typeroots-and-types)
中可以配置描述文件根目录，不过不配置也行，默认就包含了 **./node_modules/@types/**
目录。

安装完描述文件项目之后，如果文件中 import 了 node 的模块，那也能看到补全提示了。
除了 node 之外，其他的第三方包，只要提供描述文件，也都可以做到。像 lodash 的描述文件，
各种参数的描述都非常良心

![补全提示](http://crazydogs.github.io/images/typescript2.png)

当然 VSCode 的功能还有很多，包括 task 啊，各种 extension，就不说了。

## 强类型

最近其实稍微有些迷茫，感觉自己这两年，还是太局限于 JavaScript 了，不知不觉中，
即使写其他的东西，还是会带上 JS 的感觉。JS 可以说并没有什么严格的类型或者说数据结构，
平时业务上可能也没有这种复杂度，导致对这些东西，太不熟悉了，也有点抗拒，
还是要多尝试一些别的东西吧。
