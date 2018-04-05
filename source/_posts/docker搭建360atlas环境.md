title: docker搭建360atlas环境
date: 2018-4-5 10:37:12
tags:
- dataloader
- resolver
-

categories:
- 运维
---

今天我们继续来完成我们的测试环境，[上一篇](http://blog.kazaff.me/2018/04/04/docker%E6%90%AD%E5%BB%BAmysql%E4%B8%BB%E4%BB%8E%E5%BB%B6%E8%BF%9F%E5%90%8C%E6%AD%A5%E7%8E%AF%E5%A2%83/)我们完成了一个mysql主从延迟同步的环境搭建。今天我们在它的基础上，再通过docker镜像搭建一个360atlas即可。

360的这个atlas中间件已经很久没有更新了，最近看到和它相关的新闻是美团的一款基于它的[改造版](https://tech.meituan.com/dbproxy-pr.html)，推荐有需要的同学优先考虑。

回到主题，我们既然要用docker来搭建，那么就需要快速找一个它的镜像，我好像只找到了一个：[docker-360Atlas](https://github.com/mybbcat/docker-360Atlas)。那也只能用它了，不要以为我不懂感恩，是这个镜像真的有点糙啊！好歹给一个运行命令的文档啊。。。让我这种小白怎么快乐的开箱即用？！不过还好它有Dockerfile，可以看到明细。

结合360官方给的atlas安装说明（也是tm糙的一笔），我开始运行容器：

```
docker run -it --name 360atlas -v /c/Users/kazaff/atlas/conf:/usr/local/mysql-proxy/conf -p 1234:1234 -p 2345:2345 mybbcat/docker-360atlas /bin/bash
```

注意这里我们需要在容器启动后立刻链接到容器内部，因为上面这个命令是无法成功启动atlas服务进程的。。。因为镜像的设定是去加载`/usr/local/mysql-proxy/conf/docker-atlas.cnf`配置文件，但是偏偏在镜像中把这个配置文件声明成了文件夹。。。草，这tm可折腾死我了。docker无法把文件映射到文件夹。。。我也不能。

好吧，我们只能在容器内部手动启动atlas进程，并使用我们创建的`test.cnf`：

```
/usr/local/mysql-proxy/bin/mysql-proxyd test start
```

配置文件细节我就不贴了，这部分官方文档还是解释的够可以的。

至此为止，我们的环境已经搭建完成了，项目代码测试走起~~
