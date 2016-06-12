title:  关于MariaDB和mysql5.7的json类型特性
date: 2016-03-04 09:37:00
tags:
- mysql
- Mariadb
- json
categories: 数据库
---


## mysql5.7

之前有仔细的了解并使用过MongoDb，大概在一两年前吧~但无奈记忆早已模糊！
<!--more-->
最近可能有需要解决一个数据结构问题，刚好比较符合文档型数据库的领域范畴。就在我正翻看以前记录的文章时，突然想起来，似乎mysql5.7开始支持json类型，心里琢磨，如果可以避免项目中引入过多的依赖，这无疑是最明智的选择。

GG一下，刚好找到了一个入门的[文章](http://www.bytetown.net/2016/03/01/001-mysql-5_7_json_functions.html)，基本上把常用操作介绍的非常清楚了。

如果你想知道mysql5.7对json特性的实现细节，不妨看看[这里](http://mysql.taobao.org/monthly/2016/01/03/)，这样我们就可以开始尝试在业务中使用json类型啦！

虽然看文档中也提到了，目前可以针对json内部数据进行索引以及检索，但似乎没有mongodb提供的查询强大，但优势是沿用了SQL的知识，可以很快上手！

关于mysql5.7，先告一段落。

## mariadb10.1.10

我们再来看看社区版的mariadb，它从5.3版本开始就已经支持json了，不过和mysql的方法不太一样，它基于“Dynamic Columns”思路来实现的，底层和mysql方法一样都是blob类型存储。

目前来看，mariadb支持的json特性并没有mysql的多，或者说稍微有点复杂。官方资料：[Dynamic Columns](https://mariadb.com/kb/en/mariadb/dynamic-columns/)。

尤其是在处理json的嵌套时，使用的方法比较烧脑。


## todo

虽然目前不管是你选择mysql还是mariadb，都可以使用json类型来处理非结构化数据模型，但你的开发语言提供的db库是否跟得上节奏，这就是个疑问了？

目前项目主要想在数据结构模型上能获得更大的灵活性，但针对非结构数据类型的检索性能并不是非常敏感，更多的是想持久化“文档概念”的类型！所以不出意外的话，将会暂时不考虑mongodb啦~

听起来，数据库领域的革命还在激烈的进行着啊！
