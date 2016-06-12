title: 关于redis集群和事务
date: 2014-11-1 09:37:12
tags: 
- redis
- 一致性
- 原子性
- 回滚
- twemproxy
- tomcat
- 集群
- pipe
- 缓存
categories: nosql
---


最近为了核算项目的两个架构指标（可用性和伸缩性），需要对项目中使用的Redis数据库的集群部署进行一定程度的了解，当然顺便再学习一遍它的事务细节。
<!-- more -->
既然我在上面把Redis称之为数据库，那么在我们目前的项目里，它自然就需要持久化相关的数据，而不仅仅充当缓存而已！

在网上逛了一遍，看了不少关于redis集群搭建的文章，有一些把redis的主备当集群来讲的，也有一些讲的是以第三方代理方式搭建集群的，比较新的是讲的redis3.0beta提供的服务器端实现集群的~~

比较有代表型的一款中间代理方式实现redis集群的开源产品是Twitter推出的[twemproxy](https://github.com/twitter/twemproxy)。而这种方式的优劣，redis的作者早已经写过一篇分析的[文章](http://www.oschina.net/translate/twemproxy-a-twitter-redis-proxy)了，相信大家读过以后就能了解其中的好坏。

在这里，我主要是贴一下关于[twemproxy对redis命令的支持情况](https://github.com/twitter/twemproxy/blob/master/notes/redis.md)的细节，相信有了这个数据，我们在设计使用redis时可以起到指导的作用。

另外，twemproxy中不少命令的支持与否需要依赖[Hash Tags](https://github.com/twitter/twemproxy/blob/master/notes/recommendation.md#hash-tags)，简单粗暴解释的话，其实就是说twemproxy支持指定key中的某一部分用来做hash散列，这样就有助于把相关的一些数据分布在同一台服务器上，从而避免一些指令导致数据在多服务器之间不必要的迁移而影响了性能！

从这个表格中我们注意到，twemproxy不支持redis的事务操作，原因其实在上面给出的文章中已经解释了~这里我主要想来聊一下redis的事务到底是什么？记忆中看过一篇文章，模糊记得redis的事务和传统关系型数据库的事务是存在差异的。

先看一下这篇[文章](http://redisbook.readthedocs.org/en/latest/feature/transaction.html)，写的已经非常之详细了，不是么？我们必须搞清楚： **原子性、一致性和回滚功能**。这可能显得有一些过于纠结定义了，不过查了一下GG才发现，其实关于原子性和一致性的理解却是有很多种说法~~而我更偏向于下面这种解释：

- **一致性：**如果事务启动时数据是一致的，那么当这个事务成功结束的时候数据库也应该是一致的;
- **原子性：**事务的所有操作在数据库中要么全部正确反映，要么全部不反映。

分别举例子来说，一致性就是数据库的数据状态符合数据库所描述的业务逻辑和规则，比如最简单的一条一致性规则：银行账户存款余额不能是负值。

而原子性就是其字面解释：要么都执行，要么都取消！这时候就需要数据库事务有**回滚能力**。不难理解吧？

接下来我们再说redis的事务，上面的资料中提到：

	当事务失败时，Redis 也不会进行任何的重试或者回滚动作。

也就是说，redis不具备事务原子性（非事务下的redis命令具备原子性）。看一个代码例子：

	set name kazaff0  	//首先我们为key为name的键设置了一个值：kazaff0
	multi				//开启事务
	set name kazaff1	//事务里我们修改name的值为kazaff1
	lpush name kazaff2	//故意造成一个执行错误
	exec				//提交事务
	get name			//？

可以猜出最后一条指令的返回结果应该：`kazaff1`。为什么？因为redis不支持事务失败后回滚！

但是需要注意的是，服务器在执行事务之前会先检查客户端的状态，如果发现不满足事务执行的条件的话服务器端会直接终止事务，也就是说任务队列中的指令一条都没有执行！

为什么要注意这一点呢？也就是说**只有执行错误才会需要回滚，而watch，discard，入队错误等都不需要回滚**，因为执行队列中的指令压根一条都没有执行过！

以前总是把redis的事务和[pipe](http://blog.csdn.net/freebird_lb/article/details/7778919)看成一个东西：打包执行指令~~但现在才发现，完全两码事儿嘛！！





---

参考：
[理解事务的一致性和原子性](http://blog.csdn.net/amghost/article/details/17651891)







