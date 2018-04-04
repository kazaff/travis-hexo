title: docker搭建mysql主从延迟同步环境
date: 2018-4-4 10:37:12
tags:
- dataloader
- resolver
-

categories:
- 运维
---

最近打算为项目的数据库进行优化，考虑读写分离。目前打算再搭建一个中间件Atlas，放在mysql服务器前面，这样数据库的读写分类就不会被项目业务代码感知到。当然，也不是完全感知不到，一些业务还是需要强制读主库的。

考虑到Atlas提供的`/*master*/`语法我不是百分百确定，因为项目用到各种语言的mysql库，我觉得有必要测试一下在所有语言下这个特性都能得到正确的执行。基于这个原因，我决定要手动搭建一个测试环境。

这个测试环境主要是要模拟主从同步延迟，我们就靠这个延迟的时间窗口，来判断到底程序中的逻辑是与主库还是从库打交道。可能有其它的更便利的方案来完成这个判断，但我确实不知道，有知道的同学可以留言教教我。

不过，搭建这个环境的过程中还是收获不少呢（坑真多）！下面就分享给大家~

首先，我在自己的工作机上来模拟环境，意味着我要在window10系统中来搭建整个环境，为了便利，我会使用docker来分别安装主从mysql节点。GG了一下相关的文章，还是相当多的，我选择了mysql官方提供的[docker镜像](https://hub.docker.com/_/mysql/)，按照[简书的这篇分享](https://www.jianshu.com/p/ab20e835a73f)来完成主从设置。起初觉得一切都很顺利哟。然而一切都是tm的假象！

由于我使用的是mysql官方镜像，它的设置mysql配置的方式和参考文章中的不太一致，我按照官方规范来自定义mysql配置文件：

```
docker run --name mysql_master -v /d/atlas/master:/etc/mysql/conf.d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

由于我用的是window10下的docker，所以我无法像教程中那样自然的把本地文件(`/d/atlas/master`)映射到容器里，注意我这里故意写的不是 **系统用户路径**，如果你上来就使用的系统用户路径，那你可能会绕过这个坑哟~，但我觉得谁都不会把项目代码直接丢在系统用户目录下吧？！

无法映射，意味着mysql容器无法自定义，GG了一下，发现是因为win环境下需要手动在虚拟主机上做一下文件共享设置，详细步骤可以看[segmentfault上的一个提问](https://segmentfault.com/q/1010000009469821/a-1020000009484599)。

好了，第一个坑搞定了。

别高兴太早哟，我继续按照教程启动mysql容器，一切正常，但发现在主节点上执行`show master status;`这个命令无法看到结果，登录到容器内部执行`service mysql status`命令发现报警告：

```
Warning: World-writable config file '/etc/mysql/conf.d/config-file.cnf' is ignored
```

原因是mysql的安全机制认为配置文件的权限分配的太大，直接将其忽略了。。。。继续GG，试了好几个方案都没有搞定。就在我决定切换到linux环境下从头再来的时候，在[stackoverflow](https://stackoverflow.com/questions/35367935/mysql-in-docker-on-windows-world-writable-files-ignored)看到了一个简单粗暴的解决方案：

```
//启动容器
$ docker run --name mysql_slave -v /c/Users/youyou/atlas/slave:/kazaff -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 -d my
sql:latest

// 进入容器内部执行下面命令
cp /kazaff/config-file.cnf /etc/mysql/conf.d/
    && chmod 644 /etc/mysql/conf.d/config-file.cnf
```

这样我们就曲线救国式的创建了一个满足mysql安全要求的配置文件。第二个坑搞定了。

按照教程我执行`START SLAVE`命令，发现从库的状态始终是`Slave_IO_Running=NO`，查了一圈，网上各种解决方案，但我却不知道哪个适合我。我连上节点查看mysql的erro.log也没有看到对应的错误信息。最后竟然发现，执行`SHOW SLAVE STATUS`命令的结果里，其实包含了错误信息：`Last_IO_Error`，`Last_SQL_Error`。一看就明白了，原来我主从节点配置文件里的`server_id`写重复了（其实是因为从节点中错误导入了主节点的配置文件）。第三个坑搞定了。

继续按照教程做，直至结束。我们就得到了一个实时同步的主从环境。

接下来我们链接到从库上，执行：

```
STOP SLAVE;
CHANGE MASTER TO MASTER_DELAY=30;
START SLAVE;
```

我们目的达到了，现在主从节点之间同步数据会有个30秒的延迟时间窗口，接下来我们就可以来搭建Atlas节点了。放在下一篇文章继续吧~~嘿嘿嘿
