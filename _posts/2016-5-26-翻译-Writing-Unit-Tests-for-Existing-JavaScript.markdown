---
layout: post
title:  "翻译《Writing Unit Tests for Existing JavaScript》"
date:   2016-5-26 00:00:10
categories: jekyll update
---

[原文地址](https://rmurphey.com/blog/2014/07/13/unit-tests)

我们在[Bazaarvoice](http://www.bazaarvoice.com/careers/)的团队花了很多的时间思考
如何提高项目的质量，以及我们如何对我们的代码能按期望的运行有充足的信心。

我们使用功能测试很长时间了，测试类似『当一个用户点击一个按钮的时候，组件会正常运
行么』这样的问题。这些测试当然一定程度上反映了产品的质量，但是我们发现，他们异常
地脆弱。就算我们把依赖的 CSS 选择器抽象出来，测试的运行时间还是太长了，特别是须
要在所有想兼容的浏览器跑一遍。测试本身的质量问题，也很严重。部分测试确实只是单元
测试，并没有测试任何功能性的东西。

几个月前，我们团队来了一个 QA ，带领我们重新聚焦质量问题。有一个能够对质量问题考
虑如此周全、系统的同事，确实是完全不同。他不是只关心新功能是不是实现了，而是仔细
分析了我们的质量控制方案，加以改进。

他一直强调的一件事(need to push our tests down the stack，不知道怎么译)。功能测
试应该是一个黑盒子，编写并不需要知道我们的程序是怎么运作的。而单元测试，须要对实
际的代码提供大范围而且详细的覆盖。理想情况下，功能测试可以是少量的比较慢的，因为
它们充当的是一个低频率的烟雾测试(smoke test)，单元测试一股脑该是充分彻底，而且运
行起来很快的，因为我们会一直运行单元测试。

直到现在，我们的单元测试只关注通用和框架代码，比如，我们是否正确地解析了一个 URL
，而不是一些顶层的功能，比如让组件做一些事情。我试图告诉自己，这挺好的，但实际上
这是因为我害怕把功能测试加到质量参差不齐的功能性代码，特别是老代码。

在大约一个星期前，在我的同事们的威逼利诱下，我决定开始开始这项艰巨的工程，看看这
坑到底能有多深。一个星期后，我感觉像是在马拉松里面走了最开始的 10 步。当然，这些
这些起步工作让我决定继续做下去，而且也是一些提前的训练。从这方面来说，我还是已经
走了不少路了。以下是目前为止我已经完成和学到的东西。

### Step 0

幸运的是我并不是完全从零开始的，但如果你的项目里面之前并没有用过任何单元测试框架
，也不用害怕，这个并不困难。我们在 Grunt 中使用 Mocha 作为我们的测试框架，还有
expect.js 作为我们的断言库。如果我们是从现在开始的话，应该会考虑使用 Intern。

我们对单元测试进行了分组。每组包括了几个文件，每个文件测试一个 AMD 模块。在我开
始这项工作的时候，大部分被测试的模块都是低耦合的，没有太多的依赖，互动也很少。大
部分的单元测试文件都是加载一个模块，运行这个模块，检查返回结果，很简单。

但是功能性的代码，特别是老的功能性代码，情况就完全不同了。视图使用了模板，model
要获取数据，并与 view 通信。有些 model 还有继承关系。几乎所有东西都依赖一个全局
的消息中介传递信息。

因为这些代码之前都没有写过测试，可以肯定它的可测性会有很大问题。但为了可测性而进
行完全的重构肯定也是不切实际的。我们采用渐进式的重构，一点一点来，但这么做会带来
很大的风险。所以大部分时间，我们的目标会是为现有的东西写一个测试，当测试完成后，
再小心地重构这一部分。

我们决定从 model 开始下手，所以我找到一个最简单的 model

    define([
        'framework/bmodel',
        'underscore'
    ], function (BModel, _) {
        return BModel.extend({
            options : {},
            name : 'mediaViewer',

            init : function (config, options) {
                _.extend(this.options, options);
            }
        });
    });

为啥我们会有一个这样几乎啥都不做的模块呢？虽然有原因但我不打算解释，不过这给我们
讨论的问题提供了一个非常简单的开始。

我创建了一个新的测试组用来做 model 的测试，然后加入一个文件来开始写测试。我一开
始的时候天真的以为我只要加载模块，然后写几个断言就能轻松搞定，果然还是想太多。

### 依赖模拟(原文为 mock)：Squire.js

我之前在这个项目还有之前的项目里写过一些其他测试，我知道我须要去模拟一些依赖。比
如说，我们有一个叫做 ENV 的模块用来 ...，算了，知道对它有依赖就行了。虽然 ENV 里
面的大部分内容都不会被一个特定模块使用，但是基本上每个每个 model 和 view 都要依
赖它。

Squire.sj 是一个非常棒的库，用来在 RequireJS 基础上做模拟。他让你能够重写依赖的
对象。当一个 module 引用 ENV 的时候，你可以使用 Squire，说：『嘿，在这个测试里面
用我手工生成的这个对象当做依赖就好啦』。

我写了一个注入模块，用来加载 Squire，同时模拟了一些在 Node 上运行测试时会缺失的
依赖。

    define([
        'squire',
        'jquery'
    ], function (Squire, $) {
        return function () {
            var injector;

            if (typeof window === 'undefined') {
                injector = new Squire('_BV');

                injector.mock('jquery', function () {
                    return $;
                });

                injector.mock('window', function () {
                    return {};
                });
            }
            else {
                injector = new Squire();
            }

            return injector;
        };
    });

接下来我开始编写测试，看看不模拟任何东西我们的测试能走多远。我们注意到，下面这个
测试模块的主体并没有直接加载我们须要测试的东西。首先，它调用 injector 函数来初始
化依赖模拟，然后使用 injector 来引用我们要测试的模块。像普通的 require 一样，
injector.require 是异步的，我们须要让测试框架等到模块加载完成再去执行断言。

define([
    'test/unit/injector'
], function (injector) {
    injector = injector();

    var MediaViewer;

    describe('MediaViewer Model', function () {
        before(function (done) {
            injector.require([
                'bv/c2013/model/mediaViewer'
            ], function (M) {
                MediaViewer = M;
                done();
            });
        });

        it('should be named', function () {
            var m = new MediaViewer({});
            expect(m.name).to.equal('mediaViewer');
        });

        it('should mix in provided options', function () {
            var m = new MediaViewer({}, { foo : 'bar' });
            expect(m.options.foo).to.equal('bar');
        });
    });
});

上面这个测试模块当然还是会失败的。model 实例化的时候，是带着一个组件的，同时model 
也须要能够访问 ENV 因为 ENV 有组件相关的信息。创建一个『真正』的组件和让『真正』
的 ENV 包含它的信息是一个艰巨的任务，不过这正是依赖模拟要做的。

在这里一个『真正』ENV 模块是一个通过自定义配置数据实例化的 Backbone 模块，然而一
个简单得多的 ENV 模块对测试来说已经足够了。

    define([
        'backbone'
    ], function (Backbone) {
        return function (injector, opts) {
            injector.mock('ENV', function () {
                var ENV = new Backbone.Model({
                    componentManager : {
                        find : function () {
                            return opts.component;
                        }
                    }
                });
    
                return ENV;
            });
    
            return injector;
        };
    });

同样的，一个『真正』的组件是非常复杂的，但他的一部分，测试时须要的那部分是非常有
限的。最后我们模拟的组件会是像这样

define([
    'underscore'
], function (_) {
    return function (settings) {
        settings = settings || {};

        settings.features = settings.features || [];

        return {
            trigger : function () {},
            hasFeature : function (refName, featureName) {
                return _.contains(settings.features, featureName);
            },
            getScope : function () {
                return 'scope';
            },
            contentType : settings.contentType,
            componentId : settings.id,
            views : {}
        };
    };
});

在这两个模拟的模块中，我们走了一些捷径：真正的 hasFeature 方法是非常复杂的，但是
模拟的模块中，这个方法直接返回一个测试很容易获取的值。同样的，componentManager 
的 find 方法本来是非常复杂的，但在模拟的 EVN 中，这个方法一直返回同一个对象。我
们的模拟对象是设计来给用到他们的测试的，可配置，又容易预见其行为。

知道须要模拟什么，以及何时、如何去模拟，是一项须要学习的技能。模拟得不好的话，单
元测试没问题但功能完全不行这种情况也是有可能出现的。在我们项目里面，组件代码还是
有一些不错的测试，但 ENV 就不行了。我们须要解决这个问题，然后模拟 ENV 的时候就方
便多了。

目前，我的方法是这样的：先尝试不模拟任何东西让测试通过，不行的话就模拟一些，不过
越少越好。同时还要把所有的模拟依赖放到一起，这样就不用每次写测试都造轮子了。

最后，我把注入模块写成了一个共享的通用模块，任何测试都可以使用它。这样其实不好，
因为违反了『只模拟须要模拟的东西』这个大原则。下面这个注入模块返回一个函数，这个
函数能创建一个新的注入器，而不是直接返回注入器本身。

最后 MediaViewer 的测试会像是这样

define([
    // This properly sets up Squire and mocks window and jQuery
    // if necessary (for running tests from the command line).
    'test/unit/injector',
    
    // This is a function that mocks the ENV module.
    'test/unit/mocks/ENV',
    
    // This is a function that mocks a component.
    'test/unit/mocks/component'
], function (injector, ENVMock, component) {
    injector = injector();
  
    // This will become the constructor for the model under test.
    var MediaViewer;
  
    // Create an object that can serve as a model's component.
    var c = component();
  
    // We also need to mock the ENV module and make it aware of
    // the fake component we just created.
    ENVMock(injector, { component : c });
  
    describe('MediaViewer Model', function () {
        before(function (done) {
            injector.require([
                'bv/c2013/model/mediaViewer'
            ], function (M) {
                MediaViewer = M;
                done();
            });
        });
  
        it('should be named', function () {
            var m = new MediaViewer({
                component : c
            }, {});
            expect(m.name).to.equal('mediaViewer');
        });
  
        it('should mix in provided options', function () {
            var m = new MediaViewer({
                component : c
            }, { foo : 'bar' });
  
            expect(m.options.foo).to.equal('bar');
        });
    });
});

