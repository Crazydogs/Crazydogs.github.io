---
layout: post
title:  "node Mysql 库 Bookshelf"
date:   2016-8-11 16:44:10
category: coding
---

拼 SQL 是一件非常痛苦的事情，加上本身对数据库不是很熟，这两天感觉快把自己写挂了。
今天在找 node 下使用事务的方案的时候发现了 Bookshelf 这个库，试用了一下非常方便
而且个人感觉 API 设计得很简洁。

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

#### 查询

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

#### 新增/修改
model.save

````
     new Book({name: 'book name', author: 'someone'}).save();   // 新增
     Book.where({name: 'book name'}).save({author: 'another}, {method: 'update'}); 修改
````

#### 删除
model.destroy

````
    Book.where({name: 'book name'}).destroy();
````

## 概念
大概上手之后，来稍微了解一下 Bookshelf 中的概念吧
### model
model 其实是对数据库行的一种抽象，定义了库名还有与其他 model 之间的关系。model 提
供了操作数据库的简单方法，比如 fetch，count，save, where 等, 从而屏蔽了数据库层
面的具体细节。
### collection
collection 是 model 的一个有序集合，通常可以通过 model.fetchAll() 来获得。

## 返回值
Bookshelf 中的异步操作返回值都是 Promise 对象，提供了一种统一的模式，区别在于有点
Promise 对象的返回是 collection 有的是 model, 而有的操作看名字像异步但其实并不返
回 Promise，不熟悉的时候还是需要多看看接口文档。

## 简单例子
比如说我们有两张表，book 和 author，我们假定书和作者是多对一的关系。
数据库中的内容是这样的

````
    mysql> select * from books;
    +----+-----------------------+-----------+
    | id | name                  | author_id |
    +----+-----------------------+-----------+
    |  1 | 百年孤独              |         1 |
    |  2 | 霍乱时期的爱情        |         1 |
    |  3 | 东方快车杀人案        |         2 |
    |  4 | ABC 谋杀案            |         2 |
    +----+-----------------------+-----------+

    mysql> select * from author;
    +----+-------------------------+
    | id | name                    |
    +----+-------------------------+
    |  1 | 加西亚·马尔克斯         |
    |  2 | 阿加莎.克里斯蒂         |
    +----+-------------------------+
````

那我们可以通过下面简单的代码来描述这两个 model 的关系

````
    let knex = require('knex')(dbconfig);
    let bookshelf = require('bookshelf')(knex);

    let Author = bookshelf.Model.extend({
        tableName: 'author',
        book: function () {
            return this.hasMany(Book);
        }
    });
    let Book = bookshelf.Model.extend({
        tableName: 'books',
        author: function () {
            return this.belongsTo(Author);
        }
    });
````

确定关系之后，关联的数据库查询就会变得非常方便了

````
    Book.where({id: 2}).fetch({
        withRelated: 'author'
    }).then(model => {
        console.log('book');
        console.log(model.toJSON());
        console.log('author');
        console.log(model.related('author').toJSON());
    });


    // 控制台输出如下：
    // book
    // { id: 2,
    //   name: '霍乱时期的爱情',
    //   author_id: 1,
    //   author: { id: 1, name: '加西亚·马尔克斯' } }
    // author
    // { id: 1, name: '加西亚·马尔克斯' }

````

那如果假定书和作者之间存在着多对多的关系，也是没有问题的
我们在数据库中加多一张表

````
    mysql> select * from books_author;
    +-----------+---------+
    | author_id | book_id |
    +-----------+---------+
    |         1 |       2 |
    +-----------+---------+
````

然后重新定义这两个 model

````
    let Author = bookshelf.Model.extend({
        tableName: 'author',
        book: function () {
            // 如果命名按照规范的话 belongsToMany 只需要第一个参数就够了
            return this.belongsToMany(Book, 'books_author', 'author_id', 'book_id');
        }
    });
    let Book = bookshelf.Model.extend({
        tableName: 'books',
        author: function () {
            return this.belongsToMany(Author, 'books_author', 'book_id', 'author_id');
        }
    });
````

使用和之前一样的查询语句的结果差不多，只不过获得的 author 是一个 collection

````
    // 控制台输出如下：
    // book
    // { id: 2,
    //   name: '霍乱时期的爱情',
    //   author_id: 1,
    //   author: 
    //    [ { id: 1,
    //        name: '加西亚·马尔克斯',
    //        _pivot_book_id: 2,
    //        _pivot_author_id: 1 } ] }
    // author
    // [ { id: 1,
    //     name: '加西亚·马尔克斯',
    //     _pivot_book_id: 2,
    //     _pivot_author_id: 1 } ]
````




其实这东西现在还没用顺手，未完待续吧。
看了别人的文档之后，顿时感觉自己封装的 API 弱爆了，还是要学习一个。
