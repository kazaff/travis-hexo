title: Open-Falcon初探
date: 2016-08-10 09:37:12
tags:
- 监控
- docker
categories: 运维
---

随着项目的开发一点一点的完成，离初版上线日期已经越来越近了，这样就涉及到各种运维问题，监控的意义就体现出来了。虽然项目最终会部署在云平台，而云平台自身会带监控套件，不过不够灵活，一些想要的指标和报警方式还是需要自己来实现。大概看了几款监控解决方案，对Open-Falcon特别有好感，虽然我不懂GO语言~
<!--more-->

## 环境搭建

废话不多说，先来快速搭建一套Open-Falcon环境吧，官方提供了docker镜像来让我们这种新手快速尝鲜，不过是放在docker.hub的仓库中，速度不太理想。幸好有前辈将镜像clone到[灵雀](https://hub.alauda.cn/repos/lixiaowei/open-falcon)的仓库了，你懂的。

成功把镜像下载到本地后，我们还需要根据[官方提示](https://github.com/frostynova/open-falcon-docker/blob/master/README.md)，用自带的`run.sh`来实例化镜像。不过在那之前，我们要对该脚本做一下修改：

```
#!/bin/sh

HOST_DIR=/home/kazaff/open-falcon/data
DOCKER_DIR=/home/work/open-falcon

docker run -td --name open-falcon -v $HOST_DIR/conf:$DOCKER_DIR/conf -v $HOST_DIR/data:$DOCKER_DIR/data -v $HOST_DIR/logs:$DOCKER_DIR/logs -v $HOST_DIR/mysql:$DOCKER_DIR/mysql -p 8433:8433 -p 6030:6030 -p 5050:5050 -p 8088:8080 -p 8081:8081 -p 6060:6060 -p 5090:5090 -p 6081:6081 index.alauda.cn/lixiaowei/open-falcon:latest
```

一共修改了3个地方：

- 将`HOST_DIR`地址修改到了我指定的位置，为了方便后面的测试，不用来回切换路径
- 增加了`6081`端口的绑定，用于查看judge的状态，参考[这里](http://book.open-falcon.org/zh/faq/alarm.html)
- 修改了镜像的名字，毕竟我们使用的灵雀云

剩下的步骤就按照官方提示来做就ok了。


## Agent安装

docker镜像是不自带agent的，agent是安装在被监控机器的，我们可以从官方提供的[二进制安装包](http://pan.baidu.com/s/1eR1cNj8)中获取到编译好的agent。

由于我们是将docker的宿主机当作是被监控机器，所以基本上我们不需要修改agent默认的配置文件，直接根据[文档](http://book.open-falcon.org/zh/install_from_src/agent.html)提示的步骤运行即可。

目前为止，agent已经开始每分钟上报一次监控数据了。


## 架构图

![](https://raw.githubusercontent.com/open-falcon/doc/master/screenshots/falcon-arch.png)

推荐大家还是先好好读一下[官方介绍](http://book.open-falcon.org/zh/intro/index.html)，对后面可能遇到的使用问题帮助。


## 端口监控

Open-Falcon默认就会采集目标机器的200多项指标，基本上可以覆盖目标机器的OS运行状态。不过，对我们自己的应用来说，可能还不够，我们需要实时掌握应用是否还存活，所以我们需要对指定的端口号进行监控。

默认Open-Falcon是不会监控端口的，毕竟它也不能知道我们目标端口的配置。所以我们需要自己定义监控策略。

在后台中配置好用户组和用户关系后，还需要配置`HostGroup`，并为其分配目标host和策略模版才能完成我们的目标，操作流程[官方](http://book.open-falcon.org/zh/usage/getting-started.html)已描述。

一切就绪后，请求judge的http接口来查看一下配置好的策略是否生效了：

> http://judge的ip:6081/strategy/endpoint名字/net.port.listen

其中，endpoint名字根据上面的agent配置，你需要在自己docker的宿主机上执行`hostname`命令来获取。


## 坑

#### 监控表达式不起作用

我们可以在后台看到，不仅能根据HostGroups来配置策略达到监控的目的，Open-Falcon还提供了“expressions”模块，我们可以直接来进行监控配置，如下图：

![](http://pic.yupoo.com/kazaff/FLqj3EIn/wIAte.png)

我这边实测情况是，只成功了一次，然后就再也无法被judge获取到。不知道为啥？我重新安装环境都没解决~

#### 重启docker容器后hostname不识别

这个问题指的是，在Dashboard后台的endpoint搜索，无法搜索出重启容器之前的hostname节点。

只有手动编辑agent的配置文件中hostname的值，而且还不能和之前的名字一致，使用设置的名字才可以检索到结果。

而且更奇葩的是，重启前的hostname数据肯定还在，因为监控报警中还能看到针对之前名字的警告在实时更新。


#### 报警邮件无法收到

默认是没有提供邮件server的，所以我们必须自己来实现一个提供http调用的邮件服务，官方提供了一个[GO的实现](https://github.com/open-falcon/mail-provider)，但并没有给二进制版本，所以我们需要自己编译。没有GO环境的，可以直接在[这里](http://pan.baidu.com/s/1o8gvJaa)下载哟~

mail-provider的配置文件`cfg.json`的配置：
```
{
    "debug": true,
    "http": {
        "listen": "0.0.0.0:4000",
        "token": ""
    },
    "smtp": {
        "addr": "xxxx:25",
        "username": "xxxxx",
        "password": "xxxxx",
        "from": "xxxxx@xxx.xx"
    }
}
```
需要注意的是，我这里端口号用的是25，而不是465，因为实测中我使用的AWS的SMS服务，是提供465端口发送ssl邮件的，不过这个插件貌似不行，最终使用25端口才能跑正常。此外，AWS的SMS服务还需要对from设置进行邮箱校验，注意哟~

最后记得要修改一下sender的配置文件`sender.cfg`：
```
"api": {
        ......
        "mail": "http://172.17.0.1:4000/sender/mail"
    }
```
其中`api.mail`的值是用的我本地的ip。修改后执行：
```
docker exec open-falcon supervisorctl restart sender
```

#### 监控趋势图js报错

查看监控趋势图是需要选择Endpoint的，不过当你一开始跑Agent后立刻来查看趋势图，是会发现图表绘制报错的，根据我多年的js经验应该是由于没有数据导致的。

所以，稍等几分钟，待agent上报几次数据后就可以正常看到监控趋势图了。

#### 灵异

在不停地重启不停的配置不停重装环境的折腾下，还碰到了各种灵异问题，例如OpsPlatform的链接突然变成了默认设置，Agent进程莫名其妙的消失了等等。

总之感觉如果不是特别熟悉GO的话，想玩转Open-Falcon难度还是有的，风险还是大的。
