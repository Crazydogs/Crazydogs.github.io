---
layout: post
title:  "Bookshelf"
date:   2016-8-11 16:44:10
category: coding
---

拼 SQL 是一件非常痛苦的事情，加上本身对数据库不是很熟，这两天感觉快把自己写挂了。
今天在找 node 下使用事务的方案的时候发现了 Bookshelf 这个库，试用了一下非常方便
而且个人感觉 API 设计得很简介。

## 官网
[地址](http://bookshelfjs.org/)

## 安装

像官网上介绍的，npm 装上 3 个包就 ok 了

````
    $ npm install knex --save
    $ npm install bookshelf --save

    # Then add one of the following:
    $ npm install pg
    $ npm install mysql
    $ npm install mariasql
    $ npm install sqlite3
````

## 连接数据库
bookshelf 实在 knex 基础上面实现的，封装了一层便利的 API。所以初始化的时候须要先
初始化 knex，再把 knex 实例交给 bookshelf 生成 model。

````
    var knex = require('knex')({
      client: 'mysql',
      connection: {
        host     : '127.0.0.1',
        user     : 'your_database_user',
        password : 'your_database_password',
        database : 'myapp_test',
        charset  : 'utf8'
      }
    });

    var bookshelf = require('bookshelf')(knex);

    var User = bookshelf.Model.extend({
      tableName: 'users'
    });
````

## 关联表
通过 bookshelf 封装之后，可以简单地定义各个 model 之间的关系，包括一对一，一对多
和多对多。

````
    var Book = bookshelf.Model.extend({
      tableName: 'books',
      summary: function() {
        return this.hasOne(Summary);
      }
    });

    var Summary = bookshelf.Model.extend({
      tableName: 'summaries',
      book: function() {
        return this.belongsTo(Book);
      }
    });
````

这段代码关联了两个表，相当于给 book 在 summary 中添加了 book_id 外键，这样后面在
查找的时候就可以方便地进行关联查找

## 事务
bookShelf 提供了一个 transaction 方法，可以在事务中进行的操作都可以接受一个 transaction
参数

````
    var Promise = require('bluebird');

    Bookshelf.transaction(function(t) {
      return new Library({name: 'Old Books'})
        .save(null, {transacting: t})
        .tap(function(model) {
          return Promise.map([
            {title: 'Canterbury Tales'},
            {title: 'Moby Dick'},
            {title: 'Hamlet'}
          ], function(info) {
            // Some validation could take place here.
            return new Book(info).save({'shelf_id': model.id}, {transacting: t});
          });
        });
    }).then(function(library) {
      console.log(library.related('books').pluck('title'));
    }).catch(function(err) {
      console.error(err);
    });
````

## 常用操作
BookShelf 中所有数据库操作都通过 Promise 封装，熟悉的话用起来是非常爽快的。

1. 查询

    fetch/fetchPage/fetchAll

````
        // select * from `books` where `ISBN-13` = '9780440180296'
        new Book({'ISBN-13': '9780440180296'})
          .fetch()
          .then(function(model) {
            // outputs 'Slaughterhouse Five'
            console.log(model.get('title'));
          });
````

    这三个方法看名字应该就能知道什么意思了，都是返回 Promise 对象，前两个方法
    Promise 的返回为 model 或者 null，而 all 返回的是一个 collection 对象，
    collection 的 models 属性包含多个 model

    where

````
    model.where('favorite_color', '<>', 'green').fetch().then(function() { //...
    // or
    model.where('favorite_color', 'red').fetch().then(function() { //...
    // or
    model.where({favorite_color: 'red', shoe_size: 12}).fetch().then(function() { //...
````

    跟 sql 里面的 where 没啥区别，但直接用对象字面量方便了很多。

2. 新增/修改
    model.save

````
     new Book({name: 'book name', author: 'someone'}).save();   // 新增
     Book.where({name: 'book name'}).save({author: 'another}, {method: 'update'}); 修改
````

3. 删除
    model.destroy

````
    Book.where({name: 'book name'}).destroy();
````


其实这东西现在还没用顺手，未完待续吧。
看了别人的文档之后，顿时感觉自己封装的 API 弱爆了，还是要学习一个。
